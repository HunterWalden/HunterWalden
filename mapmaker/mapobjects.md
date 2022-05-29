---
title: /mapobjects/
---

# Polymorphic Object Classes

When I originally wrote the program, much of it was based on, if not pulled directly, from the original Impressive Title source. This was intentional in some places, for parity between the program and Impressive Title games, but in others, it was simply to get things done quickly. It worked at the time for a quick 1:1 interface where the user inputs something and I apply it properly, but it quickly became a problem as I tried to progress.

For example, one feature that is standard of editors like this is multi-select: the ability to hold shift and select multiple objects so they can be moved as a group. When I began trying to implement this I ran into problems almost immediately; at the time, I was storing objects in separate vectors, since they were different types: 

``` cpp
// Lists of the actual objects placed in the world
vector<WorldObject>::type mPlacedObjects;
vector<WaterPlane>::type mPlacedWaterPlanes;
vector<Portal>::type mPlacedPortals;
vector<Gate>::type mPlacedGates;
vector<WorldParticle>::type mPlacedParticles;
vector<WorldLight>::type mPlacedLights;
vector<WorldBillboard>::type mPlacedBillboards;
vector<CollBox>::type mPlacedCollBoxes;
vector<CollSphere>::type mPlacedCollSpheres;
```

At the time, the objects were simple structs that I accessed the member variables of directly. They didn't have any functions, they just existed as a way for me to store the data of each object (while they have a base struct in this iteration, my very first version did not have any inheritance at all, hence the lack of casting):

``` cpp
// Universal object options
struct UniversalObject
{
	SceneNode* mNode;
	int mType;
	bool mLockY;

	UniversalObject(SceneNode* node = 0, const int& type = 0, const bool& lockY = false)
	{
		mNode = node;
		mType = type;
		mLockY = lockY;
	}
};

// World object
struct WorldObject : UniversalObject
{
	String mMesh;
	Vector3 mRotation;
	String mSound;
	String mMaterial;
	std::pair<String, bool> mObjectFileInfo;

	WorldObject(const String& mesh = "")
	{
		mMesh = mesh;
		mRotation = Vector3::ZERO;
		mSound = "";
		mMaterial = "";
		mObjectFileInfo = std::pair<String, bool>("", false);
		mType = OBJECTTYPE_OBJECT;
	}
};

// Waterplane
struct WaterPlane : UniversalObject
{
	Vector2 mScale;
	String mMaterial;
	bool mIsSolid;
	String mSound;

	WaterPlane(const String& material = "")
	{
		mScale = Vector2(1, 1);
		mMaterial = material;
		mIsSolid = false;
		mSound = "";
		mType = OBJECTTYPE_WATER;
	}
};
//... etc
```

This is very reminiscent of how the original Impressive Title source code handles input, especially .cfg files - a lot of cluttered Manager classes and bare object structures. At the time, I was essentially just using the structs as an interface between the actual object and its exportable data, so while it was messy, it did work.

However, when expanding to something like multi-select, where the user may feasibly select several objects of various types, it quickly becomes a challenge to write clean and efficient code. I could store the SceneNode[^scenenode] and type of each object, which is what I was doing to track the current selected object, but then I would need to check the type and iterate the proper vector until I found the corresponding SceneNode[^scenenode] every time I needed a pointer to the actual object instance. I could create an ObjectGroup class that stored its own vectors of each type, but that is extremely messy and still difficult to work with, as I'd have to check every single vector for objects, even if the user only happens to have, say, 2 objects of a matching type selected.

At this point I had some basic inheritance and understanding of derived classes, but I knew there had to be a better way to store and manage these objects. I started by putting each object type into its own proper class structure and file with the same member variables and basic functions. Once I started editing the code in MapManager to reflect the new class changes with proper encapsulation/set/get methods, I was able to see patterns where I could create base class functions for the derived object types to share:

<details><summary>BaseObject class</summary>
  
 <pre>
/**
* Base class containing the basic elements of a map editor object. Not to be confused with
* Objects themselves, which are meshes placed in the world (previously named WorldObjects). 
*/
class BaseObject
{
protected:

	// SceneNode of the object
	SceneNode* mNode;

	// Actual entity
	Entity* mEntity;

	// Type of object it is
	int mType;

	// If locked, the Y never changes; if unlocked, it snaps to the terrain height
	bool mLockY;

public:

	/* Main and default constructor
	*
	* The node is not provided as it is created by the individual child classes
	*/
	BaseObject(const int& type = 0, const bool& lockY = false);
	~BaseObject();

	// Equals override compares SceneNodes
	bool operator==(const BaseObject& rhs);
	bool operator!=(const BaseObject& rhs);

	/* Create an object in the scene
	*
	* BaseObject's implementation is empty since they cannot be created
	*
	* @param sceneMgr Used to create the Entity/SceneNode
	* @param id Provided by the MapManager to name the Entity
	* @param position Position to place the SceneNode (not required since objects created by the user snap to the mouse)
	*/
	virtual void create(SceneManager* sceneMgr, const int& id, const Vector3& position = Vector3::ZERO) = 0;

	/* Destroy the Entity and SceneNode
	*
	* @param sceneMgr Requires the SceneManager to destroy the actual objects
	*/
	virtual void destroy(SceneManager* sceneMgr);

	/* Get object type */
	const int getType() const;

	/* Set if Y is locked */
	void setLockY(const bool& flag);

	/* Get if Y is locked */
	const bool getLockY() const;

	/* Toggle Y lock */
	void toggleLockY();

	/* Set node position */
	virtual void setPosition(const Vector3& position);
	virtual void setPosition(const Real& x, const Real& y, const Real& z);

	/* Set node position on a specified axis (see VECTORAXIS enum) */
	virtual void setPosition(const Real& val, const int& axis);

	/* Increment position by provided value on specified axis (see VECTORAXIS enum)
	*
	* @return The new position so the GUI can update
	*/
	virtual const Real incrementPosition(const Real& val, const int& axis);

	/* Get node position */
	virtual const Vector3 getPosition() const;

	/* Get node position on a specified axis (see VECTORAXIS enum) */
	virtual const Real getPosition(const int& axis) const;

	/* Get this object's data in string form for temporary storage
	*
	* BaseObject's implementation is empty since they cannot be created
	*
	* @return A tokenized string containing all the object's attributes
	*/
	virtual String getDataString() = 0;

	/* Get this object's unique data to locate and modify it via TempStorage
	*
	* BaseObject's implementation only returns position; certain objects have an extra data string because a user may feasibly have
	* two at the same position (objects and particles)
	*
	* @return A string containing the object's unique identifiable data
	*/
	virtual String getUniqueDataString();

	/* Get this object's world file string
	*
	* BaseObject's implementation is empty since they cannot be created
	*
	* @return The string block that exports to the world file
	*/
	virtual const String getWorldFileString() const = 0;

	/* Check entity's visibility against the camera position to see if it's out of range */
	virtual void checkVisibility(const Vector3& cameraPosition);

	/* Update object's position based on passed cursor mousepicking 
	*
	* @return Y value for GUI updates (because of LockY)
	*/
	virtual const Real update(Vector3 position);

	/* Show/hide the bounding box to visually show selection */
	virtual void setSelected(bool flag);

	/* Get SceneNode the object is on */
	SceneNode* getSceneNode() const;
};
</pre>

</details>

Everything really came together, though, when I implemented polymorphism. I already had my BaseObject class that all objects were inherited from, so now all I had to do was override base functions in the derived classes as necessary. This was particularly helpful for two reasons: for one, I could finally store all of my objects within one vector:

``` cpp
// Lists of the actual objects placed in the world
vector<BaseObject*>::type mPlacedObjects;
```

and two, I could easily grab/edit universal data via the SelectedObject (a BaseObject pointer):

``` cpp
void MapManager::setSelectedObjectLockY(bool flag)
{
	mSelectedObject->setLockY(flag);
}

const bool MapManager::getSelectedObjectLockY()
{
	return mSelectedObject->getLockY();
}

void MapManager::toggleSelectedObjectLockY()
{
	mSelectedObject->toggleLockY();
}
```

as well as unique data from any object with a simple cast rather than a vector search:

``` cpp
void MapManager::setSelectedObjectMesh(const String& mesh)
{
	static_cast<Object*>(mSelectedObject)->setMesh(mSceneMgr, mesh);
}

const String MapManager::getSelectedObjectMesh()
{
	return static_cast<Object*>(mSelectedObject)->getMesh();
}
```

One specific object this helped immensely with is CollBoxes. These are invisible collision areas that must be placed manually by the player, as the original Impressive Title does not support automatic collision. When constructed in Impressive Title, they consist of two Vector3s (x, y, z) defining the center position and outward range of the box. In my program, however, I needed to create actual box Entities[^entity] in the scene so the user could both see and interact with them. Because of the way the position/range is calculated in the original game, I found that I had to place the Entity[^entity] up half its scale on the Y axis in order for it to match up properly. This means that while every other object type's position is derived from its SceneNode[^scenenode], the CollBox Y position must be calculated using its scale. With my previous code, I had CollBox-specific position functions:

``` cpp
void MapManager::setSelectedCollBoxY(const Real& y)
{
	CollBox* tSelectedCollBox = getSelectedCollBox();
	if (!tSelectedCollBox) return;

	tSelectedCollBox->mExportedY = y;

	// Set node position based on scale
	const Vector3 tPosition = tSelectedCollBox->mNode->getPosition();
	tSelectedCollBox->mNode->setPosition(tPosition.x,
		y + (tSelectedCollBox->mSize.y / Real(2.0)),
		tPosition.z);
}

const Real MapManager::getSelectedCollBoxY()
{
	CollBox* tSelectedCollBox = getSelectedCollBox();
	if (!tSelectedCollBox) return 0.0;

	return tSelectedCollBox->mExportedY;
}
```

and I had to check the object type any time the Y position was being edited to call the proper function:

``` cpp
// Object position Y input
else if (mInputManager->getInputBox() == mObjectPositionYInputText)
{
	// CollBox Y is set different
	if (mCollBoxAttributesBox->isVisible())
	{
		mMapManager->setSelectedCollBoxY(StringConverter::parseReal(mInputManager->getInputText()));
	}
	else
	{
		mMapManager->setSelectedObjectPositionY(StringConverter::parseReal(mInputManager->getInputText()));
	}

	// If this object's Y is not locked, lock it
	if (!isCheckBoxTrue(CHECKBOX_OBJECTLOCKY))
	{
		toggleCheckBox(CHECKBOX_OBJECTLOCKY);
	}
}
```

Now, it's as easy as overriding the BaseObject position functions in my CollBox class:

``` cpp
void CollBox::setPosition(const Vector3& position)
{
	// Save passed Y
	mExportedY = position.y;

	// Set node position based on scale
	mNode->setPosition(position.x,
		mExportedY + (mScale.y / Real(2.0)),
		position.z);
}
  
const Vector3 CollBox::getPosition() const
{
	// Use exported Y
	return Vector3(mNode->getPosition().x, mExportedY, mNode->getPosition().z);
}
```

and polymorphism takes care of the rest.

<!-- Footnotes -->

[^scenenode]: SceneNodes are the main structure used to render objects in scenes in the Ogre3D rendering engine. Any MovableObject (Entity[^entity], Particle, Light, etc) must be attached to a SceneNode to appear in the scene.

[^entity]: Entities are the structure used to render 3D objects in the Ogre3D rendering engine. Entities are created with .mesh files that contain all the model's vertex, material, and skeleton information.
