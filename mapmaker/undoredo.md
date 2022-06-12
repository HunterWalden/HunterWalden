---
title: Undo & Redo
---

Among the many standard features of an editor program, the ability to undo and redo any changes is crucial for a stress-free user experience. As I had some users bug-testing the first version, this was one of the first suggestions that came up. It had been in the back of my mind but I also knew it would be a big undertaking, especially with the way my code was currently structured. Luckily, after my [huge overhaul of the object classes](/mapobjects.html), the project became much easier to visualize and implement.

I knew I needed to log each action before it was performed so that I could grab and store the old value, then revert to it if necessary. This list would need to function as a stack, since the last action is the first to be removed. For each entry on the stack, I would need to know the action type, the object it was performed on, and the old data attached to the action, if applicable. Then if the user pressed CTRL+Z, I would just need to grab the last action from the list and apply the old data to the object pointer based on the action type.

With this plan in mind, I began by writing out a struct to store actions:

``` cpp
// Structure used to store each action's data
struct MapAction
{
	// The object this action applies to
	BaseObject::Ptr mObject;
	// The type of action performed
	int mType;
	// Extra data this action may need
	String mData;

	MapAction(BaseObject::Ptr object = 0, const int& type = ACTION_NONE, String data = "")
	{
		mObject = object;
		mType = type;
		mData = data;
	}

	// Is this a valid action?
	const bool isValid() const
	{
		return (mType != ACTION_NONE);
	}
};
```

alongside an enum for the action types:

```cpp
// Action types for ActionLogger
enum ACTIONTYPE
{
	ACTION_NONE,

	// Universal
	ACTION_POSITION,
	ACTION_DELETE,
	ACTION_ADD,

	// Object
	ACTION_OBJECTSCALE,
	ACTION_OBJECTROTATION,
	ACTION_OBJECTMATERIAL,
	ACTION_OBJECTSOUND,
	ACTION_OBJECTMESH,

	// WaterPlane
	ACTION_WATERPLANESCALE,
	ACTION_WATERPLANEMATERIAL,
	ACTION_WATERPLANESOUND,

	// Portal
	ACTION_PORTALSCALE,
	ACTION_PORTALDESTINATION,

	// Gate
	ACTION_GATEMATERIAL,
	ACTION_GATEDESTINATION,
	ACTION_GATEDESTINATIONPOSITION,

	// Particle
	ACTION_PARTICLENAME,
	ACTION_PARTICLESOUND,

	// Light
	ACTION_LIGHTCOLOR,

	// Billboard
	ACTION_BILLBOARDSCALE,
	ACTION_BILLBOARDMATERIAL,

	// CollBox
	ACTION_COLLBOXSCALE,

	// CollSphere
	ACTION_COLLSPHERESCALE
};
```

With these in place, I was able to create a simple stack algorithm for adding actions to the list:

```cpp
void ActionLogger::logAction(BaseObject::Ptr object, const int& type, const String& data)
{
	// Don't exceed max actions
	if (mActionHistory.size() >= MAX_ACTION_HISTORY)
	{
		// Shift actions up to delete oldest one
		for (unsigned int i = 0; i < (MAX_ACTION_HISTORY - 1); i++)
		{
			mActionHistory[i] = mActionHistory[i + 1];
		}

		// Remove last action
		mActionHistory.pop_back();
	}

	// Add new action
	mActionHistory.emplace_back(object, type, data);
}
```

Now all I needed to do was start logging actions. My MapManager class already has function calls to set attributes on the selected object by type, so it was easy enough to call the action logger in each of them:

```cpp
void MapManager::setSelectedObjectScale(const Real& val, const bool& isLocked, const int& axis)
{
	Object::Ptr tSelectedObject = getSelectedObject();

	// Log previous scale
	mUndoLogger->logAction(mSelectedObject, ACTION_OBJECTSCALE,
		StringConverter::toString(tSelectedObject->getScale()));

	// Scale ratio locked, scale on all axes
	if (isLocked)
	{
		tSelectedObject->setLockedScale(val);
	}
	// Only scale on one axis
	else
	{
		tSelectedObject->setScale(val, axis);
	}

	// Store changes
	mTempStorageManager->logObjectChange(mSelectedObject, "MapManager::setSelectedObjectScale");
}

void MapManager::setSelectedObjectRotation(const Real& val, const int& axis)
{
	Object::Ptr tSelectedObject = getSelectedObject();

	// Log previous rotation
	mUndoLogger->logAction(mSelectedObject, ACTION_OBJECTROTATION,
		StringConverter::toString(tSelectedObject->getRotation()));

	tSelectedObject->setRotation(val, axis);

	// Store changes
	mTempStorageManager->logObjectChange(mSelectedObject, "MapManager::setSelectedObjectRotation");
}

// ...etc
```

Finally, I needed a function that would apply the provided data to the object based on action type (ie the function called on CTRL+Z):

```cpp
void MapManager::applyAction(const MapAction& action)
{
	BaseObject::Ptr tActionObject = action.mObject;

	// Apply action
	switch (action.mType)
	{
		// Universal object position
	case ACTION_POSITION:
		tActionObject->setPosition(action.mData);
		break;

		// Object scale
	case ACTION_OBJECTSCALE:
		getObject(tActionObject)->setScale(action.mData);
		break;

		// Object rotation
	case ACTION_OBJECTROTATION:
		getObject(tActionObject)->setRotation(action.mData);
		break;
		
// ...etc
```

This system worked well for Undo, which was originally all I had done and released in the first 2.0 alpha. However, at the time I had not logged object creations for undo, as I was worried about users accidentally deleting objects in the undo chain. This was the reason I decided to try adding Redo as well - initially it seemed too complicated to have both of them, especially with some of my other systems, but I decided that even if I could only store and redo one action, it would be better than nothing.

However, I realized very quickly as I began writing a Redo logger that it has the exact same functionality as Undo. Since the logger was already encapsulated in its own class, I decided to scrap the new class and create another instance of my existing class:

```cpp
ActionLogger* mUndoLogger;
ActionLogger* mRedoLogger;

...

mUndoLogger = new ActionLogger();
mRedoLogger = new ActionLogger();
```

As I began to try to call the logger on CTRL+Y, I realized that all I needed to do was remember the object's previous data before calling Undo, that way I could restore it on Redo. All the loggers need to do is swap the object's current and stored attribute data then pass it to the opposite logger. With this in mind, I could create an extremely simple function in ActionLogger that would swap the data:

```cpp
void ActionLogger::receiveAction(MapAction actionData)
{
	/* Since the opposite logger will revert the object to its *current* state, we need 
	to replace the data string with its current value */
	actionData.mData = actionData.mObject->getActionAttributeString(actionData.mType);

	// Call base function with modified data
	ActionLogger::logAction(actionData.mObject, actionData.mType, actionData.mData);
}
```

Now I could simply call my existing applyAction() function on the RedoLogger and it works as intended.

This system was perfect for basic attribute edits, but the goal of Redo in the first place was to undo/redo object creations, which would be a bit more complicated. At the time of writing my first ActionLogger I was still using raw pointers to store my objects and freeing the memory when the object was deleted by the user. When I added Undo, in order to log deletions I was grabbing the object's attributes as a data string and re-creating it when Undo was triggered. This started to become hard to track when passing data between the loggers, and in addition, I have another temporary storage system to retrieve progress on crash that needed access to the object data. 

With all these issues in mind, I switched to shared_ptrs to store my objects. Now I am able to keep objects in memory after they are "deleted" by the user in order to access them in the loggers and temporary storage, then they are freed when they are removed from temporary storage. To support this system, I added an IsCreated variable to my objects to track if they are actually created in the scene or not, and a recreate() function to restore them visually. Now to undo a deletion or redo a creation, I just need to call recreate() on the object's preexisting pointer:

```cpp
void BaseObject::recreate(SceneManager* sceneMgr)
{
	// Calls derived function
	create(sceneMgr, mID, mPosition);
}
```

I also realized that undoing a deletion is the same as redoing a creation, and vice versa. Because of this, I could simply swap the action type when the actions move between the loggers:

```cpp
void MapManager::transferAction(const bool& isRedo)
{
	// Assign loggers
	ActionLogger* tSender = 0;
	ActionLogger* tReceiver = 0;
	if (isRedo)
	{
		tSender = mRedoLogger;
		tReceiver = mUndoLogger;
	}
	else
	{
		tSender = mUndoLogger;
		tReceiver = mRedoLogger;
	}

	// Grab action data from sender
	MapAction tAction = tSender->popLastAction();

	// Validate
	if (tAction.isValid())
	{
		// Grab action object
		BaseObject::Ptr tObject = tAction.mObject;

		// Special case: deletions become creations when we swap loggers
		if (tAction.mType == ACTION_DELETE)
		{
			// Pass to receiver
			tReceiver->receiveAction(
				MapAction(tObject, ACTION_ADD));
		}
		// Special case: creations become deletions when we swap loggers
		else if (tAction.mType == ACTION_ADD)
		{
			// Pass to receiver
			tReceiver->receiveAction(
				MapAction(tObject, ACTION_DELETE));
		}
		else
		{
			// Pass to receiver
			tReceiver->receiveAction(
				tAction);
		}

		// Apply action
		applyAction(tAction);
	}
}
```

and my add/delete functions in applyAction simply recreate or delete the stored object pointer:

```cpp
case ACTION_ADD:
	// Recreate object
	tActionObject->recreate(mSceneMgr);

	// Readd to list
	mPlacedObjects.push_back(tActionObject);

	// Select
	setSelectedObject(tActionObject);

	// Pick up if necessary
	mIsPlacingObject = tActionObject->getIsHeld();

	break;

	// Universal object delete
case ACTION_DELETE:
	// Clear selection without resetting held state
	if (mSelectedObject)
	{
		resetSelectedObject(false);
	}

	// Destroy object
	deleteObject(tActionObject);

	break;
```

Now users can undo/redo creations and deletions without issue. The last step was to reset the Redo logger whenever the Undo chain is broken. Since they use the same class the setup for this is a bit odd, but it does work. I added an ActionLogger pointer to the class itself so I could pass the RedoLogger to the UndoLogger:

```cpp
// Store a pointer to the Redo logger in the Undo logger since
// it needs to be cleared when a new action is logged
ActionLogger* mCallbackLogger;

...

// Pass Redo to Undo because Undo needs to clear it when new actions are logged
mUndoLogger->initialize(mRedoLogger);
```

Then any time an action is logged via anything BUT the receiveAction() function, I can call reset() on the redo logger:

**ActionLogger::logAction()**
```cpp
// Action was done by the user, reset the Redo logger
if (!isFromLogger && mCallbackLogger)
{
	mCallbackLogger->reset();
}
```

In the end, my initial modular ActionLogger allowed me to add Redo functionality with minor edits to the existing system. In fact, it was only three steps: creating a new instance of the logger, editing the action type or data when swapping between loggers, and clearing the redo logger when the action chain is broken. 