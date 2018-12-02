# Using Retrofit

### About Retrofit

used to wrap the complexity of making networking requests with android. Allows mapping JSON directly to Kotlin classes.

**Builder**: creates concrete implementation of the interface

**Annotations**: notation data that determines how the API is called and how to map the resulting json into model objects.

## Installation

Inside the Project gradle replace the `ext.kotlin_version` with the following.

```gradle
// inside Project build.gradle
ext {
  kotlin_version = '1.2.71'
  retrofit_version = '2.3.0'
}
```

Inside the App gradle add the following

```gradle
dependencies {
  ...
  implementation "com.squareup.retrofit2:retrofit:$retrofit_version"
  implementation "com.squareup.retrofit2:converter-gson:$retrofit_version"
  ...
}
```

## Creating the JSON mapping to Concrete Classes

1. Create a `data class` that mirrors the returned JSON structure

```kotlin
data class PodcastResponse {
  val resultCount: Int,
  val results: List<ItunesPodcast>) {
    data class ItunesPodcast (
      val collectionCensoredNamed: String,
      val feedUrl: String,
      val artowrkUrl30: String,
      val releaseDate: String
    )
}
```

**Note** data classes do not have to mirror JSON structure 1 to 1. You can override this behavior.

2. Create the Service layer interface

```kotlin
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import retrofit2.http.GET
import retrofit2.http.Query
import retrofit2.Call

interface ItunesService {
    // define the route which will translate to https://itunes.apple.com/search?media=podcast&query={search_term}"
    // 1
    @GET("/search?media=podcast")
    // 2
    fun searchPodcastByTerm(@Query("term") term: String):
            Call<PodcastResponse>
    // 3
    companion object {
        // 4
        val instance: ItunesService by lazy {
            // 5
            val retrofit = Retrofit.Builder()
                .baseUrl("https://itunes.apple.com")
                .addConverterFactory(GsonConverterFactory.create())
                .build()
            // 6
            retrofit.create<ItunesService>(ItunesService::class.java)
        }
    }
}
```

1. This is a **Retrofit annoation**. Annotations always start with the `@ symbol`. This is a type of `function annotation`.

- Retrofit defines the common HTTP request verbs `GET, POST, PUT`. the `path` comes from the single parameter and applies to the function that immediately follows

2. `searchPodcastByTerms` takes single param via the `@Query` annotation. Tells Retro this parameter should be added as a URL query term inside the path defined by the @GET annoation.

- Always wrap the return type with the `Call` interface. This allows invoking the returned function asynchronously or synchronsly and get back a `Response` object containing the `PodcastResponse`.

3. a companion object is defined in the `ItunesService` interface.

4. instance prop of the companion object holds the one and only application wide instance of the `ItunesService`.

- This property returns a `Singleton` object.

**Singleton Objects**: objects that have a single instance for the lifetime of the application. No matter how many subsequent property accesses, it will only return one instance.

**Property delegation**: allows Kotlin to delegate the property setters and getters to a class.

- this is defined by the `by` keyword

```kotlin
class SomeClass: {
  val someProp: String by SomeDelegateClass()
}
```

And the delegate class must supply a `getValue` and a `setValue`

```kotlin
class SomeDelegateClass {
  operator fun getValue(thisRef: Any?, property: KProperty<*>):
    String {
      return "A delegated return value"
    }
  operator fun setValue(thisREf: Any?, property: KProperty<*>, value: String) { // no body required }
}
```

Addtional information can be found [here](https://kotlinlang.org/docs/reference/delegated-properties.html)

Essentially this is saying, instantiate this class using a `lazy` method. This is very simular to Swift's lazy var property where the value can be computed upon accessing it.

## Abstracting into a repository layer

Create a repository package within the project.

This will further abstract out the interface.

```kotlin
import me.mobilefirst.podplay.service.ItunesService
import me.mobilefirst.podplay.service.PodcastResponse
import retrofit2.Call
import retrofit2.Callback
import retrofit2.Response

// 0
class ItunesRepo(private val itunesService: ItunesService) {
    // 1
    fun searchByTerm(term: String, callBack: (List<PodcastResponse.ItunesPodcast>?) -> Unit) {
        // 2
        val podcastCall = itunesService.searchPodcastByTerm(term)
        // 3
        podcastCall.enqueue(object : Callback<PodcastResponse> {
            // 4
            override fun onFailure(call: Call<PodcastResponse>, t: Throwable) {
                callBack(null)
            }
            // 5
            override fun onResponse(call: Call<PodcastResponse>, response: Response<PodcastResponse>) {
                callBack(response?.body()?.results)
            }
        })
    }
}
```

0. Expects an already existing instance of the ItunesService layer

- this is call depency injection
- class doesn't care about what implements the service so long as the service conforms to the ItunesService interface definition

1. takes in the search term and a callback with the response. Returns a "Unit"

**Unit**: unit in Kotlin corresponds to the void in Java. Like void, Unit is the return type of any function that `does not return any meaningful value, and it is optional` to mention the Unit as the return type. But unlike void, Unit is a real class (Singleton) with only one instance.

2. Creates the instance and returns the `Call<T>` object. This does not start the network request. You must `enqueue` the request.
3. enqueue the task
4. handle failure
5. handle response success using callback. Results are optional values which may be nil.
