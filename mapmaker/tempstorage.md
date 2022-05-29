---
title: /tempstorage/
header: Temporary Object Storage
---

One goal of mine throughout development has been to catch any errors thrown by the rendering engine (Ogre) itself. This has always been a huge sore spot for Impressive Title: there is no proper try/catch exception setup, there is only one catch statement that encapsulates the entire main execution thread. Basically, if an error happens, it will tell you what it is, but the game will crash. While this likely wasn't a problem in the original Impressive Title, when the media/files were all set up to perfectly pair with the code, once you're trying to add your own maps with new assets, it can become a pain having to constantly relaunch the game. So I did my best to catch all errors and report them back via an ErrorBox, cancelling the failed action rather than ending the entire program.

Despite this, I did have a user report crashing when they tried to load in a .mesh file that had an error within the file itself. This was a case I had not considered, as my code made the assumption that if the mesh is displayed on the UI list, it exists and is valid, while the latter may not be the case. I plan to remedy that with another try/catch, however, the user made the suggestion of a file to temporarily hold objects that can be restored if the program closes. I had never done something like this before, and I was eager to try it.

I first began to break the problem down: I need to store a list of unsaved objects that can easily detect changes and export them to a file. I realized immediately that I could make use of pointers to automatically "detect" changes to the existing object list; I never have to make changes to the local objects themselves, I just need to export the pointers' data whenever a change is made:

``` cpp
/* Class for temporarily storing unsaved progress and exporting it to a file
* that can be retrieved in case of a crash
*/
class TempStorageManager
{
protected:

	// List of unsaved objects
	vector<UnsavedObject>::type mUnsavedObjects;
// ...etc
```

In order to save an object's data, I simply made a pure virtual getDataString() function in BaseObject that its derived classes use to return a tokenized string of all their attributes:

**BaseObject.h**
```cpp
/* Get this object's data in string form for temporary storage
*
* BaseObject's implementation is empty since they cannot be created
*
* @return A tokenized string containing all the object's attributes
*/
virtual String getDataString() = 0;
```

**Object.cpp**
```cpp
String Object::getDataString()
{
	String tDataString = StringConverter::toString(mType) + ";";
	tDataString += StringConverter::toString(getPosition()) + ";";
	tDataString += Utility::toOutputString(mMesh) + ";";
	tDataString += StringConverter::toString(mScale) + ";";
	tDataString += StringConverter::toString(mRotation) + ";";
	tDataString += (!hasSound() ? NO_SOUND_KEYWORD : mSound) + ";";
	tDataString += Utility::toOutputString(mMaterial) + ";";
	tDataString += Utility::toOutputString(mFileInfo.first) + ";";
	tDataString += StringConverter::toString(mFileInfo.second);

	return tDataString;
}
```

...and so on for each object type.

Next, I need to store only <i>unsaved</i> objects, so the list must be cleared and the file deleted once the user saves. Easy enough: I just call reset() on the manager class when MapManager::exportWorldFile is called:

```cpp
void TempStorageManager::reset()
{
	mUnsavedObjects.clear();

	// Delete current file
	remove(mFileName.c_str());

	// Rename file with new timestamp
	updateFileTimestamp();
}
  ```

Finally, I needed a way for the user to import the file, should they crash or close the game without saving. This presented my first issue, as it is possible for the user to edit an <i>existing</i> object, or one that is already saved to the files, or a <i>new</i> object, or one that has been newly-created and unsaved. This distinction is important because when importing unsaved changes, we must know whether to find and edit an existing object, or simply create a new one with the provided attributes. 

The solution here was to prefix existing objects with a "unique" data string: basically, what data do we need to know about this specific object to locate it in the world? In most cases, we only need to know its type and position, as we can reasonably assume that the user won't overlay two objects of the same type. This was easily accomplished by adding a new BaseObject function:

``` cpp
String BaseObject::getUniqueDataString()
{
	return (StringConverter::toString(getPosition()) + "|");
}
```

with the pipe character being used to separate existing object data with new data. 

The exceptions to this are Objects and Particles, as certain meshes may have different offsets that require them to be placed at the same position, and particles are often layered, such as a fire containing particle instances for its flames and sparks that fly out. In these two cases, the name of the mesh/particle is also included, so I simply overrode the method:

**Object.cpp**
```cpp
String Object::getUniqueDataString()
{
	return (mMesh + ";" + StringConverter::toString(getPosition()) + "|");
}
```

**Particle.cpp**
```cpp
String Particle::getUniqueDataString()
{
	return (mName + ";" + StringConverter::toString(getPosition()) + "|");
}
```

Now I just needed to check if a line is prefixed with a UniqueDataString to tell if it's an existing object and, therefore, should be located and edited rather than created.

With the basic functionality fleshed out, I could finally get to the actual guts of the code. My approach here was to call a universal function any time the user made edits to an object, passing the selected object pointer to my TempStorageManager class. The class would then internally determine if this object already exists in its list or not: if so, the file is simply saved since the pointer accesses the changed data; and if not, the pointer is added to the list and the file saved:

```cpp
void TempStorageManager::logObjectChange(BaseObject* object)
{
	// Is this object already logged?
	for (auto obj : mUnsavedObjects)
	{
		// Yes, just save changes
		if (obj.first == object)
		{
			save();
			return;
		}
	}

	// Not logged yet, grab UniqueData and add to list
	addObject(object, object->getUniqueDataString());
	save();
}
```

This system seemed to work well until I hit one major issue: the flow of execution with retrieving UniqueDataStrings. Since I was only logging data <i>after</i> changes were made, the UniqueDataString would always match the new data. That is, if I change the position of an object that has not been saved yet (therefore needs a UniqueDataString), its UniqueDataString will contain the new position since the object was already moved. I needed to make sure I stored the value <i>before</i> any changes were made to unique values (position, object mesh, particle name) in the case that it's a newly-logged object. 

To make things simple, I just made a string variable in MapManager itself to store previous data:

```cpp
// Temporary UniqueDataString storage to remember position before unsaved changes
// (Basically needed in case the user edits the position of an object that's already saved - we
// have to remember the position before any changes are made, otherwise it'll just export the same
// position twice
String mTemporaryDataStorage;
```

Then, any time a function that changes UniqueData is called, I first store the UniqueData, then call the function that changes it, then pass the object <i>and</i> stored data to TempStorageManager for validation:

**MapManager.cpp**
```cpp
void MapManager::setSelectedObjectMesh(const String& mesh)
{
	// Check for file's existence
	if (!ResourceGroupManager::getSingleton().resourceExists(OBJECT_TYPE_NAME[OBJECT_OBJECT], mesh + ".mesh"))
		throw Ogre::Exception(Exception::ERR_FILE_NOT_FOUND, "Mesh " + mesh + ".mesh does not exist.", "MapManager::setSelectedObjectMesh");

	// Save UniqueData in case this is an existing object
	storeTemporaryData();

	static_cast<Object*>(mSelectedObject)->setMesh(mSceneMgr, mesh);

	// Store changes
	logUniqueDataChange();
}
```

logUniqueDataChange simply calls this method in TempStorageManager with the proper data:

```cpp
void TempStorageManager::logUniqueDataChange(BaseObject* object, String& prevData)
{
	// Is this object already logged?
	for (auto obj : mUnsavedObjects)
	{
		// Yes, just save changes
		if (obj.first == object)
		{
			save();
			return;
		}
	}

	// Not logged yet, add to list with provided data
	addObject(object, prevData);
	save();

	// Clear temp storage
	prevData = "";
}
```

This ensures that if the user happens to edit the UniqueData of an existing/previously-saved object, it still gets logged, and therefore imported, correctly.

With that fix in place, all that was left was the import function. This had to do one of two things: either find and update an existing object, or create one with the provided data. First, the program checks for any restore files prefixed with the loaded map's name, and if so, gives the user the option to restore the unsaved changes. If they choose to do so, then the file is opened and read in line by line, adding or updating objects as it goes. First, each line is split by the pipe character to check if it applies to an existing object. If it is an existing object then its BaseObject pointer is located and updated accordingly by checking the passed data against the pointer's attributes:

```cpp
	// Object
	case OBJECT_OBJECT:
	{
		// Grab & convert attributes
		Object* tObject = static_cast<Object*>(object);
		const String tMesh = Utility::parseOutputString(data[2]);
		const Vector3 tScale = StringConverter::parseVector3(data[3]);
		const Vector3 tRotation = StringConverter::parseVector3(data[4]);
		const String tSound = (data[5] == NO_SOUND_KEYWORD ? "" : data[5]);
		const String tMaterial = Utility::parseOutputString(data[6]);
		const MapEditor::FileInfo tFileInfo = MapEditor::FileInfo(
			Utility::parseOutputString(data[7]), 
			StringConverter::parseBool(data[8]));

		// Compare each attribute and update object if necessary
		if (tObject->getPosition() != tPosition) tObject->setPosition(tPosition);
		if (tObject->getMesh() != tMesh) tObject->setMesh(mSceneMgr, tMesh);
		if (tObject->getScale() != tScale) tObject->setScale(tScale);
		if (tObject->getRotation() != tRotation) tObject->setRotation(tRotation);
		if (tObject->getSound() != tSound) tObject->setSound(tSound);
		if (tObject->getMaterial() != tMaterial) tObject->setMaterial(tMaterial);
		if (tObject->getFileInfo() != tFileInfo) tObject->setFileInfo(tFileInfo);
	}
	break;
// ...etc
```

If it is a newly-created object, then the data is simply used to create a new instance of the proper type:

```cpp
	// Object
	case OBJECT_OBJECT:
		addObject(tObjectAttributes[2],					// Mesh name
			tPosition,						// Position
			StringConverter::parseVector3(tObjectAttributes[3]),	// Scale
			StringConverter::parseVector3(tObjectAttributes[4]),	// Rotation
			Utility::parseOutputString(tObjectAttributes[5]),	// Sound
			Utility::parseOutputString(tObjectAttributes[6]),	// Material
			StringConverter::parseBool(tObjectAttributes[8]),	// Is floating
			Utility::parseOutputString(tObjectAttributes[7]));	// cfg file name
		break;
// ...etc
```

Once the file ends, it pushes a save and the temporary file is deleted, and the user can begin editing their map as usual with their restored data.
