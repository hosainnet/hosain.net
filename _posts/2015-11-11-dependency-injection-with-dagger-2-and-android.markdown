---
layout: post
title: Dependency Injection With Dagger 2 and Android
date: 2015-11-07T14:52:44+00:00
---

Dependency Injection is one of the patterns we use in software development to achieve separation of concerns, making our code maintainable and testable.

Consider the following Activity code:

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ButterKnife.bind(this);

        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://reddit.com")
                .addConverterFactory(GsonConverterFactory.create())
                .build();

        ApiService apiService = retrofit.create(ApiService.class);

        apiService.getUser("reddit")
                        .enqueue(new Callback<User>() {
                            @Override
                            public void onResponse(Response<User> response, Retrofit retrofit) {
                                displayUserDetails(response.body().getData());
                            }
                
                            @Override
                            public void onFailure(Throwable t) {
                
                            }
                        });
    }

The code above uses [Retrofit](http://square.github.io/retrofit/) to consume the [Reddit API](https://www.reddit.com/user/reddit/about.json) through an ApiService and display some basic information about a given user.

There is a key problem with the code: the Activity knows too much. It knows how to build a Retrofit object with with a domain and a Gson Converter. It also retrieves an ApiService instance using the Retrofit object. This means that:

* Reusing ApiService is difficult: we might end up duplicating code.
* Mocking/testing is difficult: It's almost impossible to mock the behaviour of the Retrofit builder or the ApiService.
* Refactoring is difficult: If we ever wanted to move away from Retrofit and make an API request in a different way, we would have to change this code and everywhere it's used in that way.

# Dagger 2

[Dagger 2](http://google.github.io/dagger/) is a lightweight DI framework by Google. It uses a annotations processing to easily declare dependencies between classes without a lot of boilerplate code.

Consider the following revised version of our previous example, using Dagger 2:

    @Inject
    ApiService apiService;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        InjectHelper.getRootComponent().inject(this);
        ButterKnife.bind(this);

        apiService.getUser("reddit")
                .enqueue(new Callback<User>() {
                    @Override
                    public void onResponse(Response<User> response, Retrofit retrofit) {
                        displayUserDetails(response.body().getData());
                    }

                    @Override
                    public void onFailure(Throwable t) {

                    }
                });
    }

It could still do with more improvements but we see the benefits straight away: The InjectorHelper is responsible for instantiating our dependencies using an @Inject annotation, and the Activity doesn't really need to know how it happened. Any place that needs to use ApiService can now have an instance ready to be used.


## Dagger 2 Setup

### Dependencies
* Add android-apt plugin to the project's build.gradle to assist with Dagger's compile time dependency:

    build.gradle (project)

         buildscript {
             repositories {
                 jcenter()
             }
             dependencies {
                 .
                 .
                 // Assists in working with annotation processors for Android Studio.
                 classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
             }
         }
 
* Apply the plugin you've just loaded to the top of your app's build.gradle:
        
    build.gradle (app)
        
        apply plugin: 'com.neenbedankt.android-apt'

* Add the dagger and the javax annotations dependencies to your app's build.gradle:

    build.gradle (app)

        dependencies {
            .
            .
            apt 'com.google.dagger:dagger-compiler:2.0.2'
            compile 'com.google.dagger:dagger:2.0.2'
            provided 'org.glassfish:javax.annotation:10.0-b28'
        }

### Structure

To setup Dagger 2, you need to define:

* The Collaborator - Your custom controller/service that you want to inject into other classes.
* A Dagger Module - Where you instantiate your collaborators and declare and any dependencies on other collaborators.
* The Component - An interface which defines all the dagger modules and where those modules can inject collaborators into.
* An Injector helper - A couple of static methods to help you inject collaborators into your classes.

To put this stuff into context, we'll implement the Reddit example above: On launch, request a user profile, and display it on the screen.

* ApiService - The Collaborator: In our case this just a Retrofit interface which retrieves the user profile, but it could be any kind of class:
         
ApiService.java

         public interface ApiService {
             @GET("/user/{user}/about.json")
             Call<User> getUser(@Path("user") String user);
         }
 
* ServiceModule - The Dagger Module: Provides an instance of our Collaborator:
 
ServiceModule.java

         @Module
         public class ServiceModule {
         
             @Provides
             @Singleton
             public ApiService providesApiService() {
                 Retrofit retrofit = new Retrofit.Builder()
                         .baseUrl("http://reddit.com")
                         .addConverterFactory(GsonConverterFactory.create())
                         .build();
         
                 return retrofit.create(ApiService.class);
             }
         }
         
The @Module and @Provides annotations are very important, they let dagger know that this module exist!

Also Notice the @Singleton annotation. This is because our service instance does not save state so it can be reused.

* RootComponent - The Component: Defines our ServiceModule and where it can inject dependencies into (MainActivity in this case):

RootComponent.java

         @Singleton
         @Component(modules = {
                 ServiceModule.class
         })
         public interface RootComponent {
             void inject(MainActivity mainActivity);
         }
 
 
* InjectorHelper - Instantiates the dependencies in Activity/Fragments:
 
InjectHelper.java

         public class InjectHelper {
         
             private static RootComponent sRootComponent;
         
             static {
                 initModules();
             }
         
             private static void initModules() {
                 sRootComponent = getRootComponentBuilder().build();
             }
         
             public static DaggerRootComponent.Builder getRootComponentBuilder() {
                 return DaggerRootComponent.builder();
             }
         
             public static RootComponent getRootComponent() {
                 if (sRootComponent == null) {
                     initModules();
                 }
                 return sRootComponent;
             }
         
         }

### Usage

Finally, in our MainActivity, we can inject the ApiService using an @Inject annotation and a simple call to our InjectHelper:

MainActivity.java

    public class MainActivity extends AppCompatActivity {

        @Inject
        ApiService apiService;
    
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            .
            .
            InjectHelper.getRootComponent().inject(this);
            apiService.getUser("reddit")
                    .enqueue(new Callback<User>() {
                        @Override
                        public void onResponse(Response<User> response, Retrofit retrofit) {
                            displayUserDetails(response.body().getData());
                        }
            
                        @Override
                        public void onFailure(Throwable t) {
            
                        }
                    });
    
        }
    }
        

### Dependencies of Dependencies

So we've just seen how to inject Dagger dependencies into an activity, let's take this a step further. For the lack of a better example, lets take the Retrofit builder and put it in it's own service "RetrofitService" and declare a dependency with ApiService.


#### First we define our service class:

RetrofitService.java:

    public class RetrofitService {
    
        public Retrofit buildRedditRetrofit() {
            return new Retrofit.Builder()
                    .baseUrl("http://reddit.com")
                    .addConverterFactory(GsonConverterFactory.create())
                    .build();
        }
    }

#### Implement a provides method in our ServiceModule:

ServiceModule.java

    @Provides
    @Singleton
    public RetrofitService providesRetrofitService() {
        return new RetrofitService();
    }


Then we simply change our provides method for ApiService to accept a RetrofitService parameter, and Dagger will do it's magic and pass in an instance of that to your provides method so it can use it

ServiceModule.java

    @Provides
    @Singleton
    public ApiService providesApiService(RetrofitService retrofitService) {
        Retrofit retrofit = retrofitService.buildRedditRetrofit();
        return retrofit.create(ApiService.class);
    }


If ApiService was a class rather than an interface, this would allow us to pass retrofitService as a constructor argument for ApiService.


# Conclusion

Using Dagger 2 can definitely improve you code readability, and delegate the creation to your objects to a set of modules that can be easily refactored and mocked. In future posts I will explain how using the above setup can help us in TDD on Android.
 
[View the full source code on Github](https://github.com/hosainnet/AndroidDagger2Example)