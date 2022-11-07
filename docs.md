# Chunity Docs

<https://www.nime.org/proceedings/2018/nime2018_paper0024.pdf>

Chunity implemented as a [Native Audio Plugin](https://docs.unity3d.com/Manual/AudioMixerNativeAudioPlugin.html)

- is this actually used? or are plugins replaced with chuckInstance?

External Variable Types

- Primitives
- Events
- UGens: The embedding host (Unity in our case) can fetch an external UGen’s most recent samples.

libChucK

- Chuck source separated into *core* and *host*
  - TODO Ge: what is difference?
  - Core: parser, and chuck VM
  - Host: ???
    - Host can implement chuck by calling core functions.
    - Other host ideas
      - glslViewer or some other shader implementation of openGL
        - other live coding tools too?
      - Web ShucK: browser running webGL shader code + implementing chuck core (via web assembly?)
        - Ideally in same browser / editor you can both write shader code and chuck code, and maybe shuck host can directly pipe values back and forth between GPU params and chuck external/globals via callbacks, without need for OSC
        -

To address the inefficiency of including multiple ChucK VMs just to spatialize audio from multiple locations, we introduced `ChuckMainInstance` and `ChuckSubInstance`.

- `ChuckMainInstance` fetches microphone input from Unity and advances time in its VM.
- `ChuckSubInstance` has a reference to a shared `ChuckMainInstance` and fetches its output samples from an external UGen in that VM (global Gain `__dac__`, buffered = true), perhaps spatializing the result along the way. This way, many spatialized ChucK scripts can all rely on the same VM and microphone, saving some computational overhead.

**Major sections**

- Unity Audio
- Chuck Core
- Chunity Interface

**Questions**

- How is data passed from chuck to Unity via global chuck variables? [chuck --> chunity]
- How is audio data from chuck Ugens sent to the Unity Audio API (+ localization, doppler, etc?) [chunity --> Unity Audio]

**Chuck -- Chunity Interface**

- connected via Native Plugins / linked Libraries
  - <https://docs.unity3d.com/Manual/NativePlugins.html>
  - See Chuck.cs L1500 for list of DLL Fns
- TODO: sync with Ge to see what all these DLL fns actually do
  - `private static extern bool runChuckFileWithArgsWithReplacementDac( System.UInt32 chuckID, System.String filename, System.String args, System.String replacement_dac );`
    - can pass colon separated args arg0:arg1 ... can we use this for better chuck subinstance addressing?
  - what do the `getters` do exactly, why do they need TypeCallback delegates? e.g. `getChuckInt()`
    - chucksubinstance.getInt() just registers a unity c# callback via `getChuckInt()`. seems the c# value of `<variableName>` is not propagated until Chuck code calls the Chuck.IntCallback, passing the current value of `variableName` as an argument
      - when in chuck land will this callback be called?

Chuck.cs

- Singleton
- Constructor Chuck()
  - sets streaming asset path
  - initializes global callback arrays
  - sets stdout/stderr/chout/cherr callback fn delegates
  - assigns AudioMixer
- InitializeFilter()
  - calls DLL Fn `initChuckInstance(chuckID, sampleRate)`
    - TODO: Ge, what does this do in Chuck native? initialize a VM?
  - InitializeFilter() called by ChuckMainInstance on Awake()
  - returns id of the newly created chuck instance
- Declares native plugin library fns with [DllImport(...)] function attribute\
  - <https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/attributes/>

ChuckMainInstance : MonoBehaviour

- Awake()
  - initializes new chuck instance via Chuck.manager.InitializeFilter() (returns int chuckId)
  - asks audio system for buffer size, num channels
  - sets up AudioSource and AudioMixerGroup
  - setup mic
- OnAudioFilterRead()
  - calls chuck.manager.ManualAudioCallback() to advance time in VM?
    - TODO Ge: what does the extern DLL `chuckManualAudioCallback(chuckID, inbuffer, outbuffer, numchannels)` do? how does this relate to advancing time in VM? what are the buffers populated with?

ChuckSubInstance : MonoBehavior

- `SetInt()` --> chuckMainInstance.SetInt(varName, val) --> Chuck.Manager.SetInt(myChuckId, varName, val) --> DLL setChuckInt
  - TODO: ge, how does `setChuck<type>()` behave in Chuck native?
- `GetInt()`
  - registers callback fn with chuck through appropriate DLL fn. (I believe) chuck native code will then call this callback, passing the current value of `variableName`
  - from NIME paper: "The get operation requires the use of a callback because the embedding host often runs on a different thread than the audio thread"
- TODO: whats the difference between IntCallback and NamedIntCallback? When do we use NamedCallbacks? on Chunity examples page, seems we do not use named callback, the callback fn only takes 1 param: the new value of the var in question e.g. `void MyGetIntCallbackFunction( long newValue )`
  - Every chuck variable getter requires passing a callback function pointer.
  - There are 3 types of function pointers:
    - Default callback
    - Named callbacks e.g. `Chuck.NamedIntArrayCallback`
      - callback function is called in addition with the string name of the chunity global
    - callback with ID e.g. `Chuck.IntArrayCallbackWithID(int callbackID,...)`
      - when calling `chuckSubInstance.get<type>`, in addition pass an ID parameter, either tied to the ID of this particlar call, or maybe the ID of the variable you are trying to update
      - chuck audio thread will then call this callback with the same callback ID you passed to the DLL function getter
    - In most cases default callback is sufficient BUT named and ID callbacks can be useful when:
      - named: you want to use the same callback handler for say, all chuck global float arrays, but want the handler to behave differently depending on which array is being updated. use a NamedCallback! e.g.

```cs
static void FloatArrayName( string name, CK_FLOAT[] value, CK_UINT length )
{
    if (name == "array1")
        this.array1 = value;
    else if (name == "array2")
        this.array2 = value;
    ...
}
```

- callback with ID: if you want to have the same callback handler behave differently depending on an ID that you specify...
  - the ID could be anything: the ID tied to the current chuck subinstance, or some ENUM value...

```cs
static void IntID( CK_INT id, CK_INT value )
{
    if (id == (CK_INT) 1) {
        // do something
    } else if (id == (CK_INT) 2) {
        // do something different
    }
}
```

- TODO: event handling
- `Awake()`
  - assign chuckMainInstance
  - assign AudioSource
    - assign audioSource.clip = spactialCip Audio/1 what is this default?
  - run `global Gain __dac__ => blackhole; true => __dac__.buffered;`
    - TODO Ge: why do this? what does buffered attribute do?
      - How does buffering work with multiple channels?
  - UpdateSpatialize();
- OnAudioFilterRead()
  - chuckMainInstance.GetUGenSamples(`global Gain __dac__`, buffer, numfRames) --> Chuck.Manager.GetUgenSamples(chuckID, ...same params) --> DLL extern getGLobalUgenSamples(chuckID, name `__dac__`, buffer, numSamples)
- Array getters / setters

*UGen.buffered*

```
int buffered( int val );
    Set the unit generator's buffered operation mode, typically used externally from hosts that embed ChucK as a component. If
true, the UGen stores a buffer of its most recent samples, which can be fetched using global variables in the host language.
int buffered();

```

TypeSyncers
-

[Unity Audio API](https://docs.unity3d.com/Manual/Audio.html)

- AudioMixer
- AudioMixerGroup
- "ChuckSubInstanceDestination"
- "ChuckMainInstanceDestination"
- AudioSettings
- [AudioSource](https://docs.unity3d.com/Manual/class-AudioSource.html)
  - plays back AudioClip to the scene. Can be played to AudioListener or AudioMixer.
  - controls spatialization/rolloff of sound
  - sets doppler effect level
- AudioListener: seems to only be used by ChuckMainInstance in WebGL context
  - WebGL does NOT support Audio Mixers
  - See ChuckMainInstance.cs L1085, adds `ChuckListenerPosition` component to the gameObject that contains an AudioListener component, calls DLL fn `setListenerTransform()` in order to spatialization in chuck? [TODO Ge confirm?]  
  - [Audio in WebGL](https://docs.unity3d.com/Manual/webgl-audio.html)
- [AudioMixer](https://docs.unity3d.com/Manual/AudioMixer.html):
- [OnAudioFilterRead()](https://docs.unity3d.com/ScriptReference/MonoBehaviour.OnAudioFilterRead.html)
  - callback called everytime buffer of audio data is sent to filter (for buffsize = 1024 and SR=44kz this is every 20ms)
  - can change the audio data before it goes on to the next filter in the chain, or being played as an audioSource
  - called from the audio thread, NOT the main thread. so cannot access many Unity functions.
  - [Audio Filters](https://docs.unity3d.com/Manual/class-AudioEffect.html)
