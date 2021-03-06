# KTX: assets management

Utilities for management of assets and heavy resources.

### Why?

LibGDX does a good job of helping you with assets (through `AssetManager`, for example), but - as usual in case of Java
APIs - it does not use the full potential of Kotlin features. This library aims to provide Kotlin extensions and wrappers
for the existing API to make assets usage more natural in Kotlin applications.

### Guide

#### Assets

- `AssetManager.load` extension method can be used to schedule asynchronous loading of an asset. It returns an asset
wrapper, which can be used as delegate property, as well as used directly to manage the asset. Usually the asset will
not be available until `AssetManager.finishLoading` or looped `AssetManager.update` are called. You can use string file
paths or `AssetDescriptor` instances to load the asset. Usage example:
```Kotlin
// Eagerly loading an asset:
val wrapper = load<Texture>("test.png")
wrapper.finishLoading()
val texture = wrapper.asset

// Delegate field:
class Test(assetManager: AssetManager) {
  val texture by assetManager.load<Texture>("test.png")
  // Type of texture property is Texture.
}
```
- `AssetManager.getAsset` utility extension method can be used to access an already loaded asset, without having to pass
class to the manager to specify asset type. This is the preferred way of accessing assets from the `AssetManager`,
provided that they were already scheduled for asynchronous loading and fully loaded. Note that this method will fail if
asset is not loaded yet. Usage example:
```Kotlin
val texture: Texture = assetManager.getAsset("test.png")
```

- `AssetManager.loadOnDemand` is similar to the `load` utility method, but it provides an asset wrapper that loads the
asset eagerly on first get call. It will not schedule the asset for asynchronous loading - instead, it will block current
thread until the asset is loaded on the first access. Use for lightweight assets that should be (rarely) loaded only when
requested. Usage example:
```Kotlin
// Eagerly loading an asset:
val texture = by assetManager.loadOnDemand<Texture>("test.png")
// Asset will be loaded upon first `texture` usage.

// Delegate field:
class Test(assetManager: AssetManager) {
  val texture: Texture by assetManager.loadOnDemand("test.png")
  // Asset will be loaded on first `texture` access.
}
```
- `AssetManager.unloadSafely` is a utility method that attempts to unload an asset from the `AssetManager`. Contrary to
`AssetManager.unload` call, this is a graceful method that will not throw any exceptions if the asset was not even
loaded in the first place. Typical usage: `assetManager.unloadSafely("test.png")`.
- `AssetManager.unload` extension method consuming a exception handling block was added, so you can gracefully handle
exception thrown during reloading. Note that `AssetManager` can throw `GdxRuntimeException` if the asset was not loaded yet.
- `AssetManager.getLoader` and `setLoader` extension methods with reified types added to ease handling of `AssetLoader`
instances registered in the `AssetManager`.

Note: if you can use coroutines in your project, [`ktx-assets-async`](../assets-async) module provides a lightweight
coroutines-based alternative to `AssetManager` that can greatly simplify your asset loading code.

#### `Disposable`

- Null-safe `Disposable.disposeSafely()` method was added. Can be called on a nullable `Disposable?` variable. Ignores
most thrown exceptions (except for internal JVM `Error` instances, which should not be caught anyway).
- `Disposable.dispose` with an exception catch block was added. Using `asset.dispose { exception -> doSomething() }`
syntax, you can omit a rather verbose try-catch block and handle exceptions with a Kotlin lambda.
- Any `Iterable` or `Array` storing `Disposable` instances will have `dispose`, `dispose { exception -> }` and
`disposeSafely` methods that dispose stored assets ignoring any `null` elements. This is a utility for disposing
collections of assets en masse.
- All exceptions get a utility `ignore()` method that you can switch at compile time (for debugging or logging) when
needed. See `Throwable.ignore()` documentation for further details.

#### `Pool`

- `Pool` instances can be invoked like a function to provide new instances of objects. Basically, this syntax: `pool()`
has the same effect as directly calling `pool.obtain()`.
- `Pool` instances can be invoked like a one argument function to free instances of objects. Basically, this syntax:
`pool(instance)` has the same effect as directly calling `pool.free(instance)`.
- New instances of `Pool` can be easily created with Kotlin lambda syntax using `pool` method. For example, this pool
would return new instances of `Entity` once empty: `pool { Entity() }`. Since this method is inlined, you should not
worry about unnecessary extra method calls or extra objects - the `Pool` implementations are prepared at compile time.

#### `FileHandle`

- Any Kotlin string can be quickly converted to a `FileHandle` instance using `toClasspathFile`, `toInternalFile`,
`toLocalFile`, `toExternalFile` or `toAbsoluteFile`. This is basically a utility for accessing `Gdx.files.getFileHandle`
method with a pleasant Kotlin syntax.
- `file` utility function allows to quickly obtain a `FileHandle` instance. It features an optional `type` parameter
which allows to choose the `FileType`, while defaulting to the most common `Internal`.

#### `FileHandleResolver`

- `FileType.getResolver` extension method was added to quickly construct `FileHandleResolver` instances for the chosen
file types.
- `FileHandleResolver.withPrefix` extension method was added to ease decoration of `FileHandleResolver` instances with
`PrefixFileHandleResolver`.
- `FileHandleResolver.forResolutions` extension method was added to ease decoration of `FileHandleResolver` instances
with `ResolutionFileResolver`.
- `resolution` factory function was added to construct `ResolutionFileResolver.Resolution` instances with idiomatic
Kotlin syntax.

#### Loaders

- `TextAssetLoader` allows to read text files asynchronously via the `AssetManager`. Very useful if you want to leverage
`AssetManager` loading and management API for a notable number of text files.

### Usage examples

Obtaining `FileHandle` instances:
```Kotlin
import com.badlogic.gdx.Files.FileType.*
import ktx.assets.*

val fileHandle = "my/file.png".toInternalFile()

val internal = file("my/file.png")
val absolute = file("/home/ktx/my/file.png", type = Absolute);
```

Working with LibGDX `Pool`:
```Kotlin
import ktx.assets.*

val pool = pool { "String." }
val obtained: String = pool() // "String."
pool(obtained) // Returned instance to the pool.
```

Gracefully disposing assets:
```Kotlin
import ktx.assets.*

texture.disposeSafely()
music.dispose { exception ->
  println(exception.message)
}
```

Disposing collections of assets:
```Kotlin
import ktx.assets.*

val textures: Array<Texture> = getMyTextures() // Works with any Iterable, too!

textures.dispose() // Throws exceptions.
textures.disposeSafely() // Ignores exceptions.
textures.dispose { exception -> } // Allows to handle exceptions.
```

Scheduling assets loading by an `AssetManager`:
```Kotlin
import ktx.assets.*

assetManager.load<Texture>("image.png")
```

Using field delegate which will eventually point to a `Texture` (after its fully loaded by an `AssetManager`):
```Kotlin
import ktx.assets.*
import com.badlogic.gdx.assets.AssetManager

class MyClass(assetManager: AssetManager) {
  val image by assetManager.load<Texture>("image.png")
  // image is Texture == true
}
```

Immediately extracting a **fully loaded** asset from an `AssetManager`:
```Kotlin
import ktx.assets.*

val texture: Texture = assetManager.getAsset("image.png")
```

Using an asset loaded on the first getter call rather than scheduled for loading:
```Kotlin
import ktx.assets.*
import com.badlogic.gdx.assets.AssetManager

class MyClass(assetManager: AssetManager) {
  val loadedOnlyWhenNeeded by assetManager.loadOnDemand<Texture>("image.png")
  // loadedOnlyWhenNeeded is Texture == true
}
```

Unloading an asset from an `AssetManager`, ignoring exceptions:
```Kotlin
import ktx.assets.*

assetManager.unloadSafely("image.png")
```

Unloading an asset from an `AssetManager`, handling exceptions:
```Kotlin
import ktx.assets.*

assetManager.unload("image.png") { exception -> exception.printStackTrace() }
```

Managing `AssetLoader` instances of an `AssetManager`:
```Kotlin
import ktx.assets.*

// Settings custom AssetLoader:
val myCustomLoader = CustomLoader()
assetManager.setLoader(myCustomLoader) // No need to pass class.

// Accessing an AssetLoader:
val loader = assetManager.getLoader<MyAsset>()
```

Creating an `InternalFileHandleResolver`:

```Kotlin
import ktx.assets.getResolver
import com.badlogic.gdx.Files.FileType

val resolver = FileType.Internal.getResolver()
```

Decorating `FileHandleResolver` with `PrefixFileHandleResolver`:

```Kotlin
import ktx.assets.*
import com.badlogic.gdx.Files.FileType

val resolver = FileType.Internal.getResolver().withPrefix("folder/")

```

Decorating `FileHandleResolver` with `ResolutionFileResolver`:
```Kotlin
import ktx.assets.*
import com.badlogic.gdx.Files.FileType

val resolver = FileType.Internal.getResolver().forResolutions(
    resolution(width = 800, height = 600),
    resolution(width = 1024, height = 768)
)
```

A simple application that registers `TextAssetLoader`, reads a file asynchronously, prints it and terminates:

```Kotlin
import com.badlogic.gdx.ApplicationAdapter
import com.badlogic.gdx.Gdx
import com.badlogic.gdx.assets.AssetManager
import ktx.assets.TextAssetLoader
import ktx.assets.load
import ktx.assets.setLoader

class MyApp : ApplicationAdapter() {
  private val manager = AssetManager()

  override fun create() {
    manager.setLoader(TextAssetLoader())
    manager.load<String>("file.txt")
  }

  override fun render() {
    if (!manager.update()) {
      println("Loading...")
    } else {
      println(manager.get<String>("file.txt"))
      Gdx.app.exit()
    }
  }
}
```

#### Implementation tip: type-safe assets

Create an enum with all assets of the selected type. Let's assume our application stores all images in `assets/images`
folder in `PNG` format. Given `logo.png`, `player.png` and `enemy.png` images, we would create a similar enum:

```Kotlin
enum class Images {
  logo,
  player,
  enemy;

  val path = "images/${name}.png"
  fun load() = manager.load<Texture>(path)
  operator fun invoke() = manager.getAsset<Texture>(path)
  companion object {
    lateinit var manager: AssetManager
  }
}
```

Operator `invoke()` function brings asset accessing boilerplate to mininum: `enumName()`. Thanks to wildcard imports, we
can access `logo`, `player` and `enemy` enum instances directly:

```Kotlin
import com.example.Images.*

// Setting AssetManager instance:
Images.manager = myAssetManager

// Scheduling logo loading:
logo.load()

// Getting Texture instance of loaded logo:
val texture = logo()

// Scheduling loading of all assets:
Images.values().forEach { it.load() }

// Accessing all textures:
val textures = Images.values().map { it() }
```

### Alternatives

- [libgdx-utils](https://bitbucket.org/dermetfan/libgdx-utils/) feature an annotation-based asset manager implementation
which easies loading of assets (through internal reflection usage).
- [Autumn MVC](https://github.com/czyzby/gdx-lml/tree/master/mvc) is a [Spring](https://spring.io/)-inspired
model-view-controller framework built on top of LibGDX. It features its own asset management module which loads and
injects assets into annotated fields thanks to reflection.
- [Kiwi](https://github.com/czyzby/gdx-lml/tree/master/kiwi) library has some utilities for assets handling, like
graceful `Disposable` destruction methods and LibGDX collections implementing `Disposable` interface. It is aimed at
Java applications though - **KTX** syntax should feel more natural when using Kotlin.
- [`ktx-assets-async`](../assets-async) module extends this library and provides `AssetStorage`: a lightweight
coroutines-based alternative to `AssetManager`.

#### Additional documentation

- [`AssetManager` article.](https://github.com/libgdx/libgdx/wiki/Managing-your-assets)
- [`FileHandle` article.](https://github.com/libgdx/libgdx/wiki/File-handling)
