# FMOD Studio integration for Godot using C++ Modules

**A Godot C++ Module that provides an integration for the FMOD Studio API.**

---

FMOD is an audio engine and middleware solution for interactive audio in games. It has been the audio engine behind many
titles such as Transistor, Into the Breach and Celeste. [More on FMOD's website](https://www.fmod.com/).

This module exposes most of the Studio API functions to Godot's GDScript and also provides helpers for performing
common functions like attaching Studio events to Godot nodes and playing 3D/positional audio. _It is still very much a
work in progress and some API functions are not yet exposed._ Feel free to tweak/extend it based on your project's needs.

**Note:** FMOD also provides a C# wrapper for their API which is used in the Unity integration and it is possible to use the
same wrapper to build an integration for Godot in C#. However do note that this would only work on a Mono build of Godot
as C# support is required and performance might not be on the same level as a C++ integration. 

**Note:** This project is a fork of [fmod-gdextension](https://github.com/utopia-rise/fmod-gdextension) from [utopia-rise](https://github.com/utopia-rise) which is a fork of [godot-fmod-integration](https://github.com/alexfonseka/godot-fmod-integration)
This fork was designed to be able to use Fmod in web builds!


#### OS Compatibility:

This plugin is compatible with Windows, Linux, OSX, HTML 5. I have no experience with Android and iOS support so they may not be available for now.

#### Godot compatibility:

This plugin should run on any 3.1+ version of Godot (Godot 4 not supported yet and will break compatibility when it comes). 
New releases are build and tested against recent version of Godot but no breaking changes should happen.

#### Fmod compatibility matrix:

| Driver Version | Api Version |
|----------------|-------------|
|      3.0.x     |   2.00.XX   | 
|      3.1+      |   2.02.XX   |

## Installing the plugin in your project

To be written...

## Using the module

### Basic usage

```gdscript
extends Node

func _ready():
	# register listener
	Fmod.add_listener(0, self)

	# play some events
	Fmod.play_one_shot("event:/Music/Level 02", self)
```

You can look at test scenes in POC folder of [example project](https://github.com/utopia-rise/fmod-gndative-godot-example-project) to find how to use the provided methods.

### Calling Studio events

Following is an example of an event instance called manually (ie. not directly managed by the integration). 
These instances are refered by an int id, returned when created. Remember to release the instance once you're done with it.

```gdscript
func _ready():	
	# register listener
	Fmod.add_listener(0, self)
	
	# play some events
	var my_music_event = Fmod.create_event_instance("event:/Music/Level 02")
	Fmod.start_event(my_music_event)
	var t = Timer.new()
	t.set_wait_time(3)
	t.set_one_shot(true)
	self.add_child(t)
	t.start()
	yield(t, "timeout")
	Fmod.stop_event(my_music_event, Fmod.FMOD_STUDIO_STOP_ALLOWFADEOUT)
	t = Timer.new()
	t.set_wait_time(3)
	t.set_one_shot(true)
	self.add_child(t)
	t.start()
	yield(t, "timeout")
	Fmod.release_event(my_music_event)
```

### Using the integration helpers

These are helper functions provided by the integration for attaching event instances to Godot Nodes for 3D/positional audio. The listener position and 3D attributes of any attached instances are automatically updated every time you call `system_update()`. Instances are also automatically cleaned up once finished so you don't have to manually call `event_release()`.

```gdscript
# play an event at this Node's position
# 3D attributes are only set ONCE
# parameters cannot be set
FMOD.play_one_shot("event:/Footstep", self)

# same as play_one_shot but lets you set initial parameters
# subsequent parameters cannot be set
FMOD.play_one_shot_with_params("event:/Footstep", self, { "Surface": 1.0, "Speed": 2.0 })

# play an event attached to this Node
# 3D attributes are automatically set every frame (when update is called)
# parameters cannot be set
FMOD.play_one_shot_attached("event:/Footstep", self)

# same as play_one_shot_attached but lets you set initial parameters
# subsequent parameters cannot be set
FMOD.play_one_shot_attached_with_params("event:/Footstep", self, { "Surface": 1.0, "Speed": 2.0 })

# attaches a manually called instance to a Node
# once attached 3D attributes are automatically set every frame (when update is called)
FMOD.attach_instance_to_node(instanceId, self)

# detaches the instance from its Node
FMOD.detach_instance_from_node(instanceId)

# blocks the calling thread until all sample loading is done
FMOD.wait_for_all_loads()
```

### Attach to existing event, 3D positioning

Here is an example where we attach event and listener to instances. In the [example project](https://github.com/utopia-rise/fmod-gndative-godot-example-project)
you have a scene `AttachToInstanceTest` where you can play with listener position, using mouse cursor.

```gdscript
func _ready():	
	# register listener
	Fmod.add_listener(0, $Listener)
	
	# Create event instance
	var my_music_event = Fmod.create_event_instance("event:/Weapons/Machine Gun")
	Fmod.start_event(my_music_event)
	
	# attach instance to node
	Fmod.attach_instance_to_node(my_music_event, $NodeToAttach)
	
	var t = Timer.new()
	t.set_wait_time(10)
	t.set_one_shot(true)
	self.add_child(t)
	t.start()
	yield(t, "timeout")
	
	Fmod.detach_instance_from_node(my_music_event)
	Fmod.stop_event(my_music_event, Fmod.FMOD_STUDIO_STOP_IMMEDIATE)
```

### Timeline marker & music beat callbacks

You can have events subscribe to Studio callbacks to implement rhythm based game mechanics. Event callbacks leverage Godot's signal system and you can connect your callback functions through the integration.

```gdscript
# create a new event instance
var my_music_event = Fmod.create_event_instance("event:/schmid - 140 Part 2B")

# request callbacks from this instance
# in this case request both Marker and Beat callbacks
Fmod.event_set_callback(my_music_event,
	Fmod.FMOD_STUDIO_EVENT_CALLBACK_TIMELINE_MARKER | Fmod.FMOD_STUDIO_EVENT_CALLBACK_TIMELINE_BEAT)

# hook up our signals
Fmod.connect("timeline_beat", self, "_on_beat")
Fmod.connect("timeline_marker", self, "_on_marker")

# will be called on every musical beat
func _on_beat(params):
	print(params)

# will be called whenever a new marker is encountered
func _on_marker(params):
	print(params)
```

In the above example, `params` is a Dictionary which contains parameters passed in by FMOD. These vary from each callback. For beat callbacks it will contain fields such as the current beat, current bar, time signature etc. For marker callbacks it will contain the marker name etc. The event_id of the instance that triggered the callback will also be passed in. You can use this to filter out individual callbacks if multiple events are subscribed.

### Playing sounds using FMOD Core / Low Level API

You can load and play any sound file in your project directory by using the FMOD Low Level API bindings. Similar to Studio Events these instances are identified by a UUID generated in script. Any instances you create must be released manually. Refer to FMOD's documentation pages for a list of compatible sound formats. You can use Fmod.load_file_as_music(path) to stream the file and loop it or Fmod.load_file_as_sound(path) to load and play it at once.
Note that instances of file loaded as sound are automatically release by FMOD once played.

```gdscript
func _ready():
	# set up FMOD
	Fmod.add_listener(0, self)
	
	Fmod.load_file_as_music("res://assets/Music/jingles_SAX07.ogg")
	music = Fmod.create_sound_instance("res://assets/Music/jingles_SAX07.ogg")
	Fmod.play_sound(my_sound)
	
	var t = Timer.new()
	t.set_wait_time(3)
	t.set_one_shot(true)
	self.add_child(t)
	t.start()
	yield(t, "timeout")
	
	Fmod.stop_sound(my_sound)
	Fmod.release_sound(music)
	Fmod.unload_file("res://assets/Music/jingles_SAX07.ogg")
```

### Muting all event

You can mute all event using `mute_all_events`. This will mute the master bus.

```gdscript
func _ready():
	# register listener
	Fmod.add_listener(0, self)
	
	# play some events
	Fmod.play_one_shot("event:/Music/Level 02", self)
	var my_music_event = Fmod.create_event_instance("event:/Music/Level 01")
	Fmod.start_event(my_music_event)
	var t = Timer.new()
	t.set_wait_time(3)
	t.set_one_shot(true)
	self.add_child(t)
	t.start()
	yield(t, "timeout")
	Fmod.mute_all_events();
	t = Timer.new()
	t.set_wait_time(3)
	t.set_one_shot(true)
	self.add_child(t)
	t.start()
	yield(t, "timeout")
	Fmod.unmute_all_events()
```

### Pausing all events

```gdscript
func _ready():
	# register listener
	Fmod.add_listener(0, self)
	
	# play some events
	Fmod.play_one_shot("event:/Music/Level 02", self)
	var my_music_event = Fmod.create_event_instance("event:/Music/Level 01")
	Fmod.start_event(my_music_event)
	var t = Timer.new()
	t.set_wait_time(3)
	t.set_one_shot(true)
	self.add_child(t)
	t.start()
	yield(t, "timeout")
	Fmod.pause_all_events(true)
	t = Timer.new()
	t.set_wait_time(3)
	t.set_one_shot(true)
	self.add_child(t)
	t.start()
	yield(t, "timeout")
	Fmod.pause_all_events(false)
```

### Changing the default audio output device

By default, FMOD will use the primary audio output device as determined by the operating system. This can be changed at runtime, ideally through your game's Options Menu.

Here, `get_available_drivers()` returns an Array which contains a Dictionary for every audio driver found. Each Dictionary contains fields such as the name, sample rate
and speaker config of the respective driver. Most importantly, it contains the id for that driver.

 ```gdscript
# retrieve all available audio drivers
var drivers = Fmod.get_available_drivers()
 # change the audio driver
# you must pass in the id of the respective driver
Fmod.set_driver(id)
 # retrieve the id of the currently set driver
var id = Fmod.get_driver()
```

### Reducing audio playback latency

You may encounter that the audio playback has some latency. This may be caused by the DSP buffer size. You can change the value **before** initialisation to adjust it:
```gdscript
Fmod.set_dsp_buffer_size(512, 4)
# retrieve the buffer length
Fmod.get_dsp_buffer_length()
# retrieve the number of buffers
Fmod.get_dsp_num_buffers()
```

### Profiling & querying performance data

`get_performance_data` returns an object which contains current performance stats for CPU, Memory and File Streaming usage of both FMOD Studio and the Core System.

```gdscript
# called every frame
var perf_data = FMOD.get_performance_data()

print(perf_data.CPU)
print(perf_data.memory)
print(perf_data.file)
```

## Contributing

In order to be able to PR this repo from a fork, you need to add `FMODUSER` and `FMODPASS` secrets to your fork repo.  
This enables CI to download FMOD api.

## Thanks

This project is a forked from [godot-fmod-integration](https://github.com/alexfonseka/godot-fmod-integration)
from [alexfonseka](https://github.com/alexfonseka) and [fmod-gdextension](https://github.com/utopia-rise/fmod-gdextension) from [utopia-rise](https://github.com/utopia-rise). We'd like to thank them for the work they did, we simply adapted their work back to C++ Modules.  
Feel free to propose any modification using github's *pull request*. We hope you'll enjoy this driver.