# Chunity Docs
https://www.nime.org/proceedings/2018/nime2018_paper0024.pdf



**Major sections**
- Unity Audio
- Chuck Core
- Chunity Interface

**Questions**
- How is data passed from chuck to Unity via global chuck variables? [chuck --> chunity]
- How is audio data from chuck Ugens sent to the Unity Audio API (+ localization, doppler, etc?) [chunity --> Unity Audio]

**Chuck -- Chunity Interface**
- connected via Native Plugins / linked Libraries 
  - https://docs.unity3d.com/Manual/NativePlugins.html
  - See Chuck.cs L1500 for list of DLL Fns
- TODO: sync with Ge to see what all these DLL fns actually do
  - `private static extern bool runChuckFileWithArgsWithReplacementDac( System.UInt32 chuckID, System.String filename, System.String args, System.String replacement_dac );`
      - can pass colon separated args arg0:arg1 ... can we use this for better chuck subinstance addressing?
  - what do the `getters` do exactly, why do they need TypeCallback delegates? e.g. `getChuckInt()`
    -  chucksubinstance.getInt() just registers a unity c# callback via `getChuckInt()`. seems the c# value of `<variableName>` is not propagated until Chuck code calls the Chuck.IntCallback, passing the current value of `variableName` as an argument
       -  when in chuck land will this callback be called? 



Chuck.cs
- Singleton
- Constructor Chuck()
  - sets streaming asset path
  - initializes global callback arrays
  - sets stdout/stderr/chout/cherr callback fn delegates
  - assigns AudioMixer
- InitializeFilter()
  - calls DLL Fn initChuckInstance(chuckID, sampleRate)
    - TODO: Ge, what does this do in Chuck native?
  - InitializeFilter() called by ChuckMainInstance on Awake()
  - returns id of the newly created chuck instance



ChuckMainInstance : MonoBehaviour
- Awake()
  - initializes new chuck instance via Chuck.manager.InitializeFilter()
  - asks audio system for buffer size, num channels
  - sets up AudioSource and AudioMixerGroup
  - setup mic
- OnAudioFilterRead()
  - calls chuck.manager.ManualAudioCallback()



ChuckSubInstance : MonoBehavior
- `SetInt()` --> chuckMainInstance.SetInt(varName, val) --> Chuck.Manager.SetInt(myChuckId, varName, val) --> DLL setChuckInt
  - TODO: ge, how does `setChuck<type>()` behave in Chuck native?
- `GetInt()`
  - registers callback fn with chuck through appropriate DLL fn. (I believe) chuck native code will then call this callback, passing the current value of `variableName`
- TODO: whats the difference between IntCallback and NamedIntCallback? When do we use NamedCallbacks? on Chunity examples page, seems we do not use named callback, the callback fn only takes 1 param: the new value of the var in question e.g. `void MyGetIntCallbackFunction( long newValue )`
- Awake()
  - assign chuckMainInstance
  - assign AudioSource
    - assign audioSource.clip = spactialCip Audio/1 what is this default?
  - run `global Gain __dac__ => blackhole; true => __dac__.buffered;`
    - TODO Ge: why do this? what does buffered attribute do?
      - How does buffering work with multiple channels? 
  - UpdateSpatialize();
- OnAudioFilterRead()
  - chuckMainInstance.GetUGenSamples(`global Gain __dac__`, buffer, numfRames) --> Chuck.Manager.GetUgenSamples(chuckID, ...same params) --> DLL extern getGLobalUgenSamples(chuckID, name `__dac__`, buffer, numSamples)
  - 

TypeSyncers
- 


Unity Audio API
- Lookup
  - AudioMixer
  - AudioMixerGroup
    - "ChuckSubInstanceDestination"
    - "ChuckMainInstanceDestination"
  - AudioSettings
  - AudioSource
  - OnAudioFilterRead()
    - 