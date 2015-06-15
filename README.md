# android-babel-sample

[Songtive](www.songtive.com) uses Xamarin as main framework for music related applications. We chosed OS X to unify the whole development for both platforms: iOS and Android. That being said, we had to configure the packaging and obfuscating process right from OS X. For obfuscating we selected https://babelfor.net/ which just works greatly without any additional tweaks. Babel provides Task for `MSBuild` to obfuscate your app but `Tasks` doesn’t work for well for Xamarin.Studio. So you are stuck to use only custom commands or custom `Targets` in `csproj`.

This document is step by step guide how to integrate Babel into your Android APK build process. 

First of all, we have to create a bash file which starts the build process and prepares APK:

```
#!/bin/bash

cd ../BabelTest

# clean project
xbuild BabelTest.csproj /p:Configuration=Release /t:Clean

# build and sign APK
xbuild BabelTest.csproj /p:Configuration=Release /t:SignAndroidPackage
```

Run it to check if script works as expected and generates APK: `BabelTest/bin/com.companyname.babeltest-Signed.apk`

Let’s move forward and include intermediate script to obfuscate assemblies before packing APK using the following `Target` in `BabelTest.csproj`:

```
  <Target Name="Obfuscate" AfterTargets="_CopyIntermediateAssemblies" Condition="'$(Configuration)' == 'Release'">
    <Exec Command="obfuscate" WorkingDirectory="../scripts" />
  </Target>
```

Please note that this `Target` is used only in Release mode and after “_CopyIntermediateAssemblies” which basically copies all assemblies into `/BabelTest/obj/Release/assemblies`.

The `obfuscate` script is located in `./scripts` directory and has the following content:

```
#!/bin/bash
BABEL_PATH=./babel.exe
SRC=../BabelTest/obj/Release/assemblies

mono $BABEL_PATH --unicode s,_ $SRC/BabelTest.dll $SRC/BabelLibrary.dll --output $SRC/BabelTest.dll

# prepare stub assembly
mcs -t:library -out:$SRC/BabelLibrary.dll stub.cs
```

We specify to Babel that we are going to merge our library right into Android application and obfuscate them together. Since Xamarin is still trying to load `BabelLibrary` during packaging - we generate stub assembly with empty class and overwrite it. 

Let’s run /script/build again and unzip APK to get `/assemblies/BabelTest.dll`. Use ILSpy to see what we have here:

![](https://github.com/desunit/android-babel-sample/blob/master/images/ilspy.png)


Your assemblies are obfuscated. 
