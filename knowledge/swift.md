


## Constants and Variables
var x = 0.0, y = 0.0, z = 0.0

var red, green, blue: Double

let π = 3.14159
let 你好 = "你好世界"
let 🐶🐮 = "dogcow"


print("The current value of friendlyWelcome is \(friendlyWelcome)")



## Floating-Point Numbers

Double represents a 64-bit floating-point number.
Float represents a 32-bit floating-point number.

Double has a precision of at least 15 decimal digits, whereas the precision of Float can be as little as 6 decimal digits. The appropriate floating-point type to use depends on the nature and range of values you need to work with in your code. In situations where either type would be appropriate, Double is preferred.


let decimalInteger = 17
let binaryInteger = 0b10001       // 17 in binary notation
let octalInteger = 0o21           // 17 in octal notation
let hexadecimalInteger = 0x11     // 17 in hexadecimal notation


let decimalDouble = 12.1875
let exponentDouble = 1.21875e1
let hexadecimalDouble = 0xC.3p0


Numeric literals can contain extra formatting to make them easier to read. Both integers and floats can be padded with extra zeros and can contain underscores to help with readability. Neither type of formatting affects the underlying value of the literal:

let paddedDouble = 000123.456
let oneMillion = 1_000_000
let justOverOneMillion = 1_000_000.000_000_1

## Type Aliases

typealias AudioSample = UInt16


## Tuples

let http404Error = (404, "Not Found")
let (justTheStatusCode, _) = http404Error


## Optionals

let convertedNumber = Int(possibleNumber)
// convertedNumber is inferred to be of type "Int?", or "optional Int"

if convertedNumber != nil {
    print("convertedNumber contains some integer value.")
}

if let constantName = someOptional {
    statements
}
// You can use both constants and variables with optional binding. 

// You can include as many optional bindings and Boolean conditions in a single if statement as you need to, separated by commas. 
if let firstNumber = Int("4"), let secondNumber = Int("42"), firstNumber < secondNumber && secondNumber < 100 {
    print("\(firstNumber) < \(secondNumber) < 100")
}


## Error Handling

func canThrowAnError() throws {
    // this function may or may not throw an error
}



do {
    try canThrowAnError()
    // no error was thrown
} catch {
    // an error was thrown
}


## Nil-Coalescing Operator

a ?? b   

equal

a != nil ? a! : b


## Range Operators

### Closed Range Operator

The closed range operator (a...b) defines a range that runs from a to b, and includes the values a and b.

for index in 1...5 {
    print("\(index) times 5 is \(index * 5)")
}

### Half-Open Range Operator

The half-open range operator (a..<b) defines a range that runs from a to b, but doesn’t include b. 

for i in 0..<count {
    print("Person \(i + 1) is called \(names[i])")
}

### One-Sided Ranges

for name in names[2...] {
}

for name in names[...2] {
}



## String

let quotation = """
The White Rabbit put on his spectacles.  "Where shall I begin,
please your Majesty?" he asked.

"Begin at the beginning," the King said gravely, "and go on
till you come to the end; then stop."
"""



let wiseWords = "\"Imagination is more important than knowledge\" - Einstein"
// "Imagination is more important than knowledge" - Einstein
let dollarSign = "\u{24}"        // $,  Unicode scalar U+0024
let blackHeart = "\u{2665}"      // ♥,  Unicode scalar U+2665
let sparklingHeart = "\u{1F496}" // 💖, Unicode scalar U+1F496



As mentioned above, different characters can require different amounts of memory to store, so in order to determine which Character is at a particular position, you must iterate over each Unicode scalar from the start or end of that String. For this reason, Swift strings can’t be indexed by integer values.



## collection

var shoppingList: [String] = ["Eggs", "Milk"]

for item in shoppingList {
}

for (index, value) in shoppingList.enumerated() {
}

var favoriteGenres: Set<String> = ["Rock", "Classical", "Hip hop"]

var namesOfIntegers = [Int: String]()

for (airportCode, airportName) in airports {
}

for airportCode in airports.keys {

for airportName in airports.values {



if #available(iOS 10, macOS 10.12, *) {