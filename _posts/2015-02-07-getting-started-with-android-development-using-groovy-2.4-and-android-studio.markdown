---
layout: post
title: "Getting Started With Android Development Using Groovy 2.4 and Android Studio"
date: 2015-02-07T18:28:25+00:00
---

Groovy 2.4 was released last month with native support for Android development.

In this post I’ll explain how to set up a new Android Studio project with Groovy support as I couldn't find a decent tutorial to explain it for newcomers to Android development (like myself).

# Why Groovy?

If you’re new to Groovy, you may ask why would I want to do this?

In short: less code = (hopefully) less buggy and more readable code. Here are some of the benefits of using Groovy over Java:

* Syntax: with support of closures and dynamic typing, Groovy allows you to write much more readable code in fewer lines of code. Some examples:

        ListView myListView = (ListView) findViewById(android.R.id.list)
        /* Implicit typing - and no semicolons! */
        def myListView = findViewById(android.R.id.list) as ListView


        Bundle extras = getIntent().getExtras();
        /* 'get' is optional in groovy for getSomething() methods */
        def extras = intent.extras


* [Powerful collections](http://groovy.codehaus.org/Collections): creating and manipulating lists / maps is very easy and powerful out of the box (each, findAll, collect, etc), which is a good way to shorten your code.


* [Dynamic](http://groovy.codehaus.org/Dynamic+Groovy): Groovy’s metaclassing capabilities allow you to extend an existing class behaviour such as overriding or adding functionality if you like.


# Setup

So let's get started by creating a new project using Android Studio. For this example I’ll use the Basic Activity template with the default settings.

You should see the following file structure created:

![Android project structure](/images/blog/2015/new-android-project.png)

The key files for us here are the two build.gradle files.

The first one is project-wide build.gradle which applies to all project modules. The second build.gradle for the app's main module.


# Add Groovy dependencies

In the app module build.gradle, we need to make two changes:

* Apply the groovy-android plugin below the line "apply plugin: 'com.android.application'" - adds support for the Groovy language in the Android Gradle toolchain.

        apply plugin: 'groovyx.grooid.groovy-android'

* Add the grooid depedency - a special version of the Groovy compiler for Android:

        dependencies {
            //..
            compile 'org.codehaus.groovy:groovy:2.4.0:grooid'
        }

The final file should look similar to this:

    apply plugin: 'com.android.application'
    apply plugin: 'groovyx.grooid.groovy-android'

    android {
        compileSdkVersion 21
        buildToolsVersion "21.1.2"

        defaultConfig {
            applicationId "net.hosain.mygroovyapplication"
            minSdkVersion 15
            targetSdkVersion 21
            versionCode 1
            versionName "1.0"
        }
        buildTypes {
            release {
                minifyEnabled false
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            }
        }
    }

    dependencies {
        compile fileTree(dir: 'libs', include: ['*.jar'])
        compile 'com.android.support:appcompat-v7:21.0.3'
        compile 'org.codehaus.groovy:groovy:2.4.0:grooid'
    }


In the project build.gradle, add the plugin dependency:

    classpath 'org.codehaus.groovy:gradle-groovy-android-plugin:0.3.5'

The final file should look like:

    // Top-level build file where you can add configuration options common to all sub-projects/modules.

    buildscript {
        repositories {
            jcenter()
        }
        dependencies {
            classpath 'com.android.tools.build:gradle:1.0.0'
            classpath 'org.codehaus.groovy:gradle-groovy-android-plugin:0.3.5'
            // NOTE: Do not place your application dependencies here; they belong
            // in the individual module build.gradle files
        }
    }

    allprojects {
        repositories {
            jcenter()
        }
    }

You should see a gradle sync message in Android Studio, click "Sync Now" and it should complete successfully.

# Groovify it!

* The groovy plugin looks for files in app/src/main/groovy, so rename main/java to main/groovy (You won't see the name change to 'groovy' in the left folder list due to the lack of Groovy support in Android Studio).

At this point, your app should run fine! This is because **you can mix and match groovy/java code**.

* Rename all files in src/groovy/\*\*/\*.java to end with '.groovy'

Now you should be able to make use of Groovy across your application.

Let's take MainActivity.groovy as an example: try removing semicolons at the end of each line and rerun the app!


# Issues and tips

* As I mentioned before, Android Studio doesn't fully support Groovy yet. For example you can't right click and create a Groovy class. You would have to create a empty file ending with '.groovy', and the annoying left hand side menu won't update to source folder name 'groovy'.

* Annotate your classes with [@CompileStatic](http://docs.codehaus.org/display/GroovyJSR/GEP+10+-+Static+compilation) - this is especially good if you're just starting out or don't need to use much of the dynamic stuff as it forces strict typing at compile time. **It also helps greatly in performance**.

        @CompileStatic
        class MainActivity extends ActionBarActivity {

        }


* ProGuard rules: Add the following to your app/proguard-rules.pro - this helps keep your app at the minimal size when you build your APKs

        -dontobfuscate
        -keep class org.codehaus.groovy.vmplugin.**
        -keep class org.codehaus.groovy.runtime.dgm*
        -keepclassmembers class org.codehaus.groovy.runtime.dgm* {
            *;
        }

        -keepclassmembers class ** implements org.codehaus.groovy.runtime.GeneratedClosure {
            *;
        }

        -dontwarn org.codehaus.groovy.**
        -dontwarn groovy**

* Gradle build task fails due to lint errors:

    If you run './gradlew clean build' in your project's root directory, it'll fail with a "InvalidPackage: Package not included in Android" lint error - [full error message](http://pastebin.com/raw.php?i=jtPwcFyE).

    It seems like some of the base Groovy classes make calls to unsupported java code which Android doesn't like.

    For now, you can suppress these errors by adding the following to the app module build.gradle inside the android block:

        android {
            ...
            lintOptions {
                disable 'InvalidPackage'
            }
        }