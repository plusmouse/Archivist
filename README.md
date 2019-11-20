# Archivist

Archivist is a flexible data storage service for WoW AddOns. It is designed especially for addons which need to store a large amount of data, but only occasionally read or update this data. Data given to the archivist is stored in SavedVariables in a compressed format, to minimize addon load time.

## Usage

Using the archivist is very simple. If you just wish to use the global archive, then install Archivist as a standalone addon. Note that if you use the global archive, it is recommended to set Archivist as a dependency in your .toc file.

First, load your data store from the archive:

```lua

-- Load data from the archive. If the RawData store MyAddonArchive doesn't exist in the archive, then it is created automatically. Usually you will use this to obtain your store.
myStore = Archivist:Load("RawData", "MyAddonArchive")

-- if you have good reason to believe that your store is archived, then you can use open:
myStore = Archivist:Open("RawData", "MyAddonArchive") -- throws an error if "MyAddonArchive" doesn't exist in the archive

-- if you know that your store doesn't yet exist, then you can call Create:
myStore = Archivist:Create("RawData", "MyAddonArchive")

-- If you wish to create an "anonymous" store for some reason, that is also supported. Archivist will auto-generate a random storeID for you. But don't lose the storeID, or it will be hard to find this data again later.
myStore, storeID = Archivist:Create("RawData")

```

All of the basic Archivist Verbs (Create, Load, Open, Close, Commit) take as parameters the Store Type and Store ID. The Store Type must be a string, and identifies the kind of store object you wish to create. The store type must be registered via RegisterStoreType (see [Create New Store Types](#create-new-store-types) below for more information). The Store ID must be a string, and is unique for that given Store Type. You may have as many stores named `MyAddonArchive` as you like, so long as all of them have differing Store Types.

The RawData store type is just a table. You may mutate this table in any way you see fit. Once you are done with your reads and writes, you may close myStore:

```lua
Archivist:CloseStore(myStore)
-- or...
Archivist:Close("RawData", "MyAddonArchive")
```

Once closed, any changes to myStore will not be archived unless you reopen the store. Depending on the store type chosen, this may.

Some store types (see [Create New Store Types](#create-new-store-types) below) may operate in such a way that it makes sense to commit changes to the archive, without closing the store. Archivist supports this operation too:

```lua
Archivist:CommitStore(myStore)
-- or...
Archivist:Commit("RawData", "MyAddonArchive") -- In the case of RawData this is not very useful, since its contents are also committed when the store is closed
```

All open stores are automatically closed and archived on `PLAYER_LOGOUT`. You should not try to read or write data from a store on or after that event. The exact behavior at that point depends on the implementation of the Store Type, as well as the order in which WoW dispatches events to addon scripts, and is outside the Archivist specification.

### Included Store Types

Archivist comes prepackaged with some basic store types, both for basic uses and as an example for implementing your own. They are listed here:

- RawData
  - A simple table with no extra bells or whistles. The contents of this table are stored directly into the archive when this archive is committed or closed.
- Store
  - Similar to RawData, but with Commit, Close, Clone functions grafted on. These are aliases to the corresponding Archivist verbs, provided for convenience.

### Embedding the Archivist

In most use cases, you will want to embed Archivist, so that your addon's archive does not intersect with that of any other addon. Embedding the Archivist into your addon is very similar, but a few more steps are needed. First, create an archive addon:

```
## Interface: 80205
## Title: MyArchive
## LoadOnDemand: 1
## SavedVariables: MyArchiveSaved
```

You may also use your addon's native SavedVariables file if you wish (instead of creating a secondary addon), but this will cause your addon to always load its archive from the disk, which may not be desirable.

Now, embed Archivist into your addon. This can be easily done via an xml file, similar to how you would embed a library:

```xml
<Ui xmlns="http://www.blizzard.com/wow/ui/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.blizzard.com/wow/ui/ ..\FrameXML\UI.xsd">
	<!-- Other embedded addons...-->
	<Include file=".\Embeds\Archivist.xml"/>
</Ui>

```

You may also load Archivist via your .toc file. It is recommended to load Archivist.xml instead of loading the lua files explicitly. This way, you do not have to worry about any directory structure changes internal to Archivist.

To use Archivist in your addon code:

```lua
local addon, ns = ...

-- Note that Archivist embeds itself in your addon's namespace.
local Archivist = ns.Archivist

-- when you have need to access the archive, then load the data...
LoadAddOn("MyArchive")
-- and initialize Archivist with your archive.
Archivist:Initialize(MyArchiveSaved)

```

Now, you may use the Archivist just like you would if it had been installed in a standalone form.

## Create New Store Types

Some use cases demand more sophisticated data management. Archivist can accomodate this need, with custom Store Types:

```lua
local prototpye = {
	id = "MyStoreType",
	version = 1,
	exclusive = false,
	Init = function() end, 
	Create = function(...) end,
	Open = function(data) end,
	Update = function(data) end,
	Commit = function(store) end,
	Close = function(store) end,
}

Archivist:RegisterStoreType(prototype)
```

Prototype Fields are as follows:

- id
  - Unique Identifier of the store type, e.g. `RawData`.
- version
  - Version number of store type. Useful if the prototype changes in a backwards incompatible way, and archived data needs to be massaged before use.
- exclusive
  - If truthy, then attempting to open the same storeID will generate an error. an exclusive store type will have a bit more safety about data updates, but be slightly less convenient to use.
  - If false (or unset), then Opening the same storeID multiple times will return the active store object that was previously generated.

Prototype methods:

- Init
  - Optional function. Initialize your prototype. If provided, then Init is always guaranteed to run exactly once per game session, before any other method is run.
- Create
  - Create a new, empty, store object. Extra arguments passed into Archivist:Create will be passed into this function if you wish to accomodate initial setup of the store object.
  - Must return non-nil store object.
- Open
  - Create from data an active store object.
  - Must return non-nil store object
- Update
  - Optional function. If provided, then archived data is replaced with the return value of Update. If no change is needed, then return nil.
- Commit
  - Return image of data to be archived
- Close
  - Deactivate store. Returned value will be written to archive. If no update to archive is needed, then return nil.
  - Once close is called on a store, Archivist will not update the archived data again unless the store is opened.

### Store Type Methods Should be Functional

With the exception of Init, all of these functions may be called at any time without warning. Thus, they should ideally be written as close to purely functional as possible, with few-to-no side effects. If you must have side effects, then do your best to write functions whose side effects are idempotent. This will help you avoid weird problems from functions being called in an order you didn't expect, and other hard-to-debug behaviors.

### Re-registering prototypes

If multiple independent codebases share an archive, and they both register the same store type, then what happens depends on the version number:

- If the second registration has equal or lower version number, then the second registration is ignored. 
- If the second registration has a higher version number, then:
  - Each active store is Closed using the old Close method
  - Each Archived store is Updated using the new Update method if provided
  - The new Init method is run.
  - Any previously open stores are opened.

Re-registering a store type is generally not recommended, as you risk data loss and other errors. If you are using a store type in a shared archive which you expect to be registered multiple times (e.g. by independent custom code auras in WeakAuras), then it is recommended to only keep stores of that type open for the time that you need them to be open. Alternatively, design the Initialization routine such that it can broadcast the re-initialization, so that any store which was previously open can be re-obtained.

## Full API List

```lua

-- Main API

-- Opens (or creates) the given store. This is the main entry point for your addon's code.
store = Archivist:Load(storeType, storeID)

-- Register store type. Store type must be registered before an archive can be accessed. All verbs will raise an error if called with an unregistered storeType.
Archivist:RegisterStoreType(prototype)


-- Archivist verbs
-- 	These are the main "actions" that archivist does.

-- Creates a new archive, and returns an active store object. Raises an error if archive already exists. In most cases you'll want to use Load instead.
store = Archivist:Create(storeType, storeID, ...)
-- or... (though rarely useful)
store, storeID = Archivist:Create(storeType)

-- Opens an archive, and returns active store object. Raises an error if archive doesn't yet exist. In most cases you'll want to use Load instead.
store = Archivist:Opent(storeType, storeID)

-- Commit to archive without closing store. This will cause a change in the archive.
-- 	Store is still considered open after committing it.
Archivist:CommitStore(store)
-- or...
Archivist:Commit(storeType, storeID)

-- Closes a store. Once closed, you may discard the store object.
-- 	May also update archive, depending on the store type. Once closed, you should consider the store object obsolete.
-- 	Occurs automatically on PLAYER_LOGOUT.
Archivist:CloseStore(store)
-- or...
Archivist:Close(storeType, storeID)
-- Note: once a store is closed, manipulating the old store object is considered outside of the Archivist specification, and behavior depends on the store type implementation. However, any changes after Close will *never* be archived


-- Miscellaneous verbs
-- 	These actions are available for your use, but are generally less useful

-- Close and delete store permanently. Use ONLY if you are absolutely sure you don't want the data anymore. Archivist cannot help you retrieve lost data once you invoke this.
-- 	If force is truthy, then the Delete will go through even if the store type is not registered. This is useful if you decide to drop support for a storeType.
Archivist:DeleteStore(store, force)
-- or...
Archivist:Delete(storeType, storeID, force)
-- or... (if you want to delete everything)
-- If storeType is given, then all stores of that type are deleted. If storeType is not given, then all stores in the entire archive are deleted. This is provided to assist in cases like, "the user wishes to destroy all data and start fresh".
-- USE WITH CAUTION. YOUR DATA WILL BE LOST. YOU WILL NOT GET A SECOND CHANCE.
Archivist:DeleteAll(storetype)

-- Create identical but independent copy of archive. If openstore is truthy, then also Opens the cloned archive and returns active store object.
store = Archivist:CloneStore(store, openStore)
-- or...
store = Archivist:Clone(storeType, storeID, openStore)


-- Plumbing Methods
-- 	The following methods are used internally, and are usually not useful for addons using the Archivist.

-- Generate a random uuid. Used when Create is called without providing a storeID
uuid = Archivist:GenerateID()

-- Compress data. Data passed in is not touched in any way, and calling code may retain ownership.
compressedString = Archivist:Archive(data)

-- Decompress data. String is expected to have been compressed using Archivist:Archive.
-- 	Compressing, and then decompressing the same data is essentially an expensive clone operation.
data = Archivist:DeArchive(compressedString)

-- Close all stores immediately. Automatically called on PLAYER_LOGOUT. Not usually useful for addons using 
Archivist:CloseAllStores()
```

## Limitations and "Gotchas"

Like every other project, Archivist can't do everything. The major restrictions (and consequences of these limitations) are described here.

### Archivist Modifies Your Addon's Namespace

Specifically, when Archivist is embedded into your addon, then the Archivist field on your namespace table (the one you access with `local addon, ns = ...`) is set to the Archivist object. I don't expect many people have ever used that particular field on their namespace table, but please do be aware of this if you decide to use the archivist.

### Archivist Can't Store Everything

Archivist is a WoW addon, and it cannot do anything that the WoW addon environment does not allow for. In practice, this means that Archivist cannot serialize functions or closures it receives, as there is no way to convert these to a string format. If your store type has functions which need to be stored (e.g. generated code), then instead of trying to archive the function directly, store all of the information which was used to generate the function, and in your Open procedure, re-generate the function based on the data image you receive.

Additionally, Archivist does not check for metatables. So, if your store type uses __index or __newindex metamethods in any way, then providing the store object when asked for a data image will likely result in data loss. To avoid this, ensure that your Close and Commit methods return objects that contain all of the data you need to archive directly, without needing to invoke any metamethods.

### Archivist is Not a Library

In the WoW addon world, we are used to embedded code that is "library-style". That is, everyone shares the same code, and shares the same versioning. If the user installs two addons, both of which use the same embedded code of different versions, then the expectation is often that both addons use the latest version of this common code. This is where tools like LibStub and the Ace framework come into play. These are great when you just want to use code that somebody else already wrote, without worrying too much about implementation details. Archivist even uses this for compression, because I don't want to have to reinvent the wheel just to compress some data. Also, LibDeflate is really good.

Archivist explicitly does not follow this pattern. In a slightly different world, Archivist might have been designed to fit the LibStub paradigm, but the specifics of the WoW AddOn environment (and what Archivist is supposed to do) make that the wrong pattern to follow, in my opinion. So, Archivist is instead designed as a service you may embed into your code, but it does not share its code or internal memory with other instances of Archivist in other addons. If a user installs two addons employ the Archivist, then you will have two independent instances of the Archivist service. These instances do not share prototype registrations, and they do not share archives (unless they were initialized with the same archive, which is also probably a bad idea). The prototypes may share the same code, if e.g. both were registered via some LibStub invocation. But if you want a particular store type, then you are responsible for registering that prototype. Don't rely on some other common addon registering the prototype for you, because it won't work.

Also, don't register Archivist with LibStub. This will "work", in that you won't get any errors. But if somebody else does the same thing, then one of you will be unable to access your own archive, since Archivist is designed to be run and initialized by each addon independently for each instance. If a user installs two addons which both use Archivist like this, then nothing will seem amiss for a while until one of these addons is disabled and suddenly the other "loses" all of its data. Then you get angry users, and nobody likes angry users.

You can set Archivist as a hard dependency in your .toc file, if you want to use a "global" archive. In this way, Archivist can behave nicely while still being "shared code". However, a global archive is like writing code with lots of global variables - not such a great idea. The standalone Archivist addon is distributed primarily so that it can be embedded into other addons easily via curseforge or packaging scripts, as well as for the use of individual users in private, non-shared setups that just need a place to store some data without too much hassle.
