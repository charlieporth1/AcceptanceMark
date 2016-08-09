Write a command line tool that can take a folder containing `.md` files specifying the tests to be run, and generates a set of XCTest-compatible Swift test classes that will be run as part of the test target(s).

The command line tool, named `amtool` will run as a pre-compilation Run Script Phase:

<img width="512" alt="amtool-run-script" src="https://cloud.githubusercontent.com/assets/153167/17513052/ac4ce036-5e22-11e6-8f10-6784d53e6e28.png">

## Example

Given an acceptance test file named `acceptancemark/image-tests.md` with the following contents:

```
## Image Loading

| name:String   || loaded:Bool  |
| ------------- || ------------ |
| available.png || true         |
| missing.png   || false        |
```

Running the command `amtool --input-dir acceptancemark` will generate a test file named `acceptancemark/ImageTests_ImageLoading.swift`, defined as:

```
struct ImageTests_ImageLoadingInput {
    let name: String
}

struct ImageTests_ImageLoadingResult: Equatable {
    let loaded: Bool
}

protocol ImageTests_ImageLoadingTestRunnable {
    func run(input: ImageTests_ImageLoadingInput) throws -> ImageTests_ImageLoadingResult
}


class ImageTests_ImageLoadingTests: XCTestCase {

    var testRunner: ImageTests_ImageLoadingTestRunnable!
    
    
    override func setUp() {
        // MARK: Implement the ImageTests_ImageLoadingTestRunner() class!
        testRunner = ImageTests_ImageLoadingTestRunner()
    }
    
    func testImageLoading_0() {
        
        let input = ImageTests_ImageLoadingInput(name: "available.png")
        let expected = ImageTests_ImageLoadingResult(loaded: true)
        let result = try! testRunner.run(input)
        XCTAssertEqual(expected, result)
    }

    func testImageLoading_1() {
        
        let input = ImageTests_ImageLoadingInput(name: "missing.png")
        let expected = ImageTests_ImageLoadingResult(loaded: false)
        let result = try! testRunner.run(input)
        XCTAssertEqual(expected, result)
    }
}

func == (lhs: ImageTests_ImageLoadingResult, rhs: ImageTests_ImageLoadingResult) -> Bool {
    return lhs.loaded == rhs.loaded
}
```

### TODO

- [x] Add **Xcode macOS Command Line Tool** target to project, named `amtool`
- [ ] Parse `--input-dir` and `-i` command line argument to specify input directory for reading `.md` files
- [ ] For each `.md` file in input directory, run test generator component

### Test Generator specification

The test generator transforms a given input file into a data structure that can be used to generate the Swift test classes. By definition, input files are specified in **Markdown** format, however the generator needs to be able to interpret only a small subset of the **Markdown** specification, namely **Headings** and **Tables**. Everything else can be ignored.

Given the following representation of a test set:

```
struct TestSpec {
	let fileName: String
	let title: String
	
	enum VariableType {
		case Bool
		case Int
		case Float
		case String
	}
	
	struct Variable {
		let name: String
		let type: VariableType
	}
	
	struct TestData {
		let inputs: [ Any ]
		let outputs: [ Any ]	
	}
	
	let inputs: [ Variable ] 
	let outputs: [ Variable ]
	
	let tests: [ TestData ]
}
```

When this sample input file is parsed:

```
## Image Loading

| name:String   || loaded:Bool  |
| ------------- || ------------ |
| available.png || true         |
| missing.png   || false        |
```

Then, the test generator should create the following `TestSpec`:

```
let inputVars = [ Variable(name: "name", type: .String) ]
let outputVars = [ Variable(name: "loaded", type: .Bool) ]
TestSpec(fileName: "image-tests", title: "Image Loading",
	inputs: inputVars,
	outputs: outputVars,
	tests: [
		TestData(inputs: [ "available.png" ], outputs: [ "true" ])
		TestData(inputs: [ "missing.png" ], outputs: [ "false" ])
	])
```

This `TestSpec` can then be used to generate the `acceptancemark/ImageTests_ImageLoading.swift` file defined above.

### Sample implementation

```
func generateTestClass(testSpec: TestSpec) {

	let fileHandle = NSFileManager.openForWriting(name: testSpec.filePath)
	
	fileHandle.writeHeader(testSpec.headerInfo)
	
	let inputVars = testSpec.inputs
	let outputVars = testSpec.outputs
	
	for i in 0..<testSpec.tests {
		let testData = testSpec.tests[i]
		let inputString = makeInputs(testData.inputs, inputVars)
		let expectedString = makeOutputs(testData.outputs, outputVars)
		let test = "func testImageLoading_0() {
       	\(inputString)
	       \(outputString)
   	   		let result = try! testRunner.run(input)
       	XCTAssertEqual(expected, result)
    	}"
    	fileHandle.write(test)
	}
}
```
