# Kotlin

## Misc
* `?`: __save call operator__ if null it returns null instead of _NullPointerException_
* `!!`: __non-null assertion operator__ throws _NullPointerException_ if null -> javalike

* Example (check if null with Elvis):
    ``` kotlin
    val nullableString: String? = // some value or null
    val length: Int = nullableString?.length ?: 0
    // 'length' will be 0 if 'nullableString' is null
    println("Length is $length")
    ```
