You can create a `job` (i.e. *build-n-test*) where the value of `strategy.matrix` is different based off of some criteria by setting the value of `strategy.matrix` to the deserialized `output` of a previous job (i.e. *matrix_prep*). This previous job would have the responsibility of constructing the `matrix` value as per your custom criteria. The following yaml demonstrates this (a copy has been included later on with comments added in for explanation):

```yaml
name: Configurable Build Matrix

on: push
jobs:
  matrix_prep:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
    - name: Check out code into the Go module directory
      uses: actions/checkout@v2
    - id: set-matrix
      run: |
        branchName=$(echo '${{ github.ref }}' | sed 's,refs/heads/,,g')  
        matrix=$(jq --arg branchName "$branchName" 'map(. | select((.runOn==$branchName) or (.runOn=="always")) )' matrix_includes.json)                
        echo ::set-output name=matrix::{\"include\":$(echo $matrix)}\"
  build-n-test:
    needs: matrix_prep
    runs-on: ${{ matrix.runs_on }}
    strategy:
      matrix: ${{fromJson(needs.matrix_prep.outputs.matrix)}}
    steps:    
    - run: echo "Hello ${{ matrix.someValue }}"
```

The `matrix_includes.json` file has the following contents:

```json
[
    {
        "runs_on":"ubuntu-16.04",
        "someValue":"Foo",
        "runOn":"always"
    },
    {
        "runs_on":"ubuntu-18.04",
        "someValue":"Bar",
        "runOn":"v2.1-Release"
    },
    {
        "runs_on":"ubuntu-20.04",
        "someValue":"Hello again",
        "runOn":"v2.1-Release"
    }
]
```

Using the setup above, one configuration will be included for *all* builds, and two will only be included if the branch name matches *v2.1-Release*. With some tweaks to the `sed` and `jq` options in the Workflow file, the branch name restrictions could be made looser such that you could have configurations run for all branches that include `-Release` (instead of just for a single branch). I may include this in this answer if there is interest (as it does not necessarily match your current question).

## `set-matrix` Job Explanation

As far as the `set-matrix` task is concerned, please refer to the following notes:

```bash
# ${{ github.ref }} returns the full git ref. As such, 'refs/heads/` should be stripped for easier future use
branchName=$(echo '${{ github.ref }}' | sed 's,refs/heads/,,g')  
# Use jq to read in a json file that represents the matrix configuration. Each block has a 'runOn' property. 
# The jq filter is setup to only output items that are set to 'always' or that have a branch name that matches 
# the current branch.
matrix=$(jq --arg branchName "$branchName" 'map(. | select((.runOn==$branchName) or (.runOn=="always")) )' matrix_includes.json)        
# This 'echo' uses a special syntax so that the output of this job is set correctly
echo ::set-output name=matrix::{\"include\":$(echo $matrix)}\"
```

## Workflow explanation

The following yaml content should be the same as above with some additional comments to help explain things:

```yaml
name: Configurable Build Matrix

on: push
jobs:
  matrix_prep:
    runs-on: ubuntu-latest
    # Defining outputs of a job allows for easier consumption and use
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
    # Checking out code as the set-matrix step utilizes a file named matrix_includes.json
    - name: Check out code into the Go module directory
      uses: actions/checkout@v2
    # This step is explained more in a following section
    - id: set-matrix
      run: |
        branchName=$(echo '${{ github.ref }}' | sed 's,refs/heads/,,g')  
        matrix=$(jq --arg branchName "$branchName" 'map(. | select((.runOn==$branchName) or (.runOn=="always")) )' matrix_includes.json)                
        echo ::set-output name=matrix::{\"include\":$(echo $matrix)}\"
  build-n-test:
    # By stating 'needs' here, the output of 'matrix_prep' is available to this job
    needs: matrix_prep
    runs-on: ${{ matrix.runs_on }}
    strategy:
      # We need to convert the json string output into an object that the GitHub Workflow expects.
      # Thankfully, the json-schema for Workflows allows 'matrix' to be set to an expression.
      matrix: ${{fromJson(needs.matrix_prep.outputs.matrix)}}
    steps:
    # Output a configuration specific value as proof and as a sanity check
    - run: echo "Hello ${{ matrix.someValue }}"
```

# Demonstration

I put together two files for this demonstration:

* The [workflow definition](https://github.com/JoshuaTheMiller/StackOverflowAnswers/blob/main/.github/workflows/configurableBuildMatrix.yml) (pasted below)
* A [JSON file to contain matrix information](https://github.com/JoshuaTheMiller/StackOverflowAnswers/blob/main/matrix_definitions.json) (pasted below)

The following two screenshots are from runs on different branches using the same workflow definition. Please note that the amount of build-n-test Jobs are different between the two:

*Build for main branch*
[![Build for main branch][1]][1]
*Build for some-Release branch*
[![Build for some-release branch][2]][2]

This is due to one build occurring on the `main` branch, and the other occurring on the `some-Release` branch. The workflow definition is configured such that branches that end in "-Release" will use an additional "ubuntu-20.04" agent (as specified in the included json file). Please note that the definitions currently included in the linked repository are slightly different than those above.

### Further Detail

#### Matrix Configuration Filtering

The filtering is accomplished through using [jq](https://stedolan.github.io/jq/) to select objects from the json array that either have their `runOn` value set to `always` or that match the current `branchName`. This is the slight tweak to your logic that I mentioned earlier: instead of saying `includeInDevBranchBuilds`, I am using `runOn` as it seemed to work better for this specific example. 

#### BranchName

The `set-matrix` step uses a value set from the previous line: `branchName=$(echo '${{ github.ref }}' | sed 's,refs/heads/.*-,,g')`. This line will strip `refs/heads/` from the branch ref and store the result in the value `branchName`. For example, if your branch is `2.1-Release`, `branchName` will be set to `2.1-Release`, and the filter from earlier will then match any objects that have `"runOn":"2.1-Release"` *or* `"runOn":"always"`.

#### The JSON file

The JSON file was created to simulate the **content** of the `includes` statement from the workflow that you linked. JSON is used as GitHub Actions have builtin JSON functions. As a sample, the following is my take on converting your `matrix:include` section to JSON. **Please note** that I've changed `includeInDevBranchBuilds` to be `runOn`, with the values set to either `always` or `v2.1-Release`.

```json
[
   {
      "displayTargetName": "centos-8",
      "os": "unix",
      "compiler": "g++",
      "runs_on": "ubuntu-latest",
      "container_image": "sophistsolutionsinc/stroika-buildvm-centos-8-small",
      "cpp_version": "c++17",
      "config_name": "Release",
      "extra_config_args": "--apply-default-release-flags --trace2file enable",
      "runOn": "always"
   },
   {
      "displayTargetName": "ubuntu-18.04-g++-8 (Debug)",
      "os": "unix",
      "compiler": "g++-8",
      "runs_on": "ubuntu-latest",
      "container_image": "sophistsolutionsinc/stroika-buildvm-ubuntu1804-regression-tests",
      "cpp_version": "c++17",
      "config_name": "Debug",
      "extra_config_args": "--apply-default-debug-flags --trace2file enable",
      "runOn": "always"
   },
   {
      "displayTargetName": "ubuntu-20.04-g++-9 (Debug)",
      "os": "unix",
      "compiler": "g++-9",
      "runs_on": "ubuntu-latest",
      "container_image": "sophistsolutionsinc/stroika-buildvm-ubuntu2004-regression-tests",
      "cpp_version": "c++17",
      "config_name": "Debug",
      "extra_config_args": "--apply-default-debug-flags --trace2file enable",
      "runOn": "v2.1-Release"
   },
   {
      "displayTargetName": "ubuntu-20.04-g++-10 (Debug)",
      "os": "unix",
      "compiler": "g++-10",
      "runs_on": "ubuntu-latest",
      "container_image": "sophistsolutionsinc/stroika-buildvm-ubuntu2004-regression-tests",
      "cpp_version": "c++17",
      "config_name": "Debug",
      "extra_config_args": "--apply-default-debug-flags --trace2file enable",
      "runOn": "v2.1-Release"
   },
   {
      "displayTargetName": "ubuntu-20.04-g++-10",
      "os": "unix",
      "compiler": "g++-10",
      "runs_on": "ubuntu-latest",
      "container_image": "sophistsolutionsinc/stroika-buildvm-ubuntu2004-regression-tests",
      "cpp_version": "c++17",
      "config_name": "Release",
      "extra_config_args": "--apply-default-release-flags --trace2file enable",
      "runOn": "always"
   },
   {
      "displayTargetName": "ubuntu-20.04-g++-10-c++2a",
      "os": "unix",
      "compiler": "g++-10",
      "runs_on": "ubuntu-latest",
      "container_image": "sophistsolutionsinc/stroika-buildvm-ubuntu2004-regression-tests",
      "cpp_version": "c++2a",
      "config_name": "Release",
      "extra_config_args": "--apply-default-release-flags --trace2file enable",
      "runOn": "v2.1-Release"
   },
   {
      "displayTargetName": "ubuntu-20.04-clang++-10",
      "os": "unix",
      "compiler": "clang++-10",
      "runs_on": "ubuntu-latest",
      "container_image": "sophistsolutionsinc/stroika-buildvm-ubuntu2004-regression-tests",
      "cpp_version": "c++17",
      "config_name": "Release",
      "extra_config_args": "--apply-default-release-flags --trace2file enable",
      "runOn": "v2.1-Release"
   }
]
```

  [1]: https://i.stack.imgur.com/looi6.png
  [2]: https://i.stack.imgur.com/7Tlvd.png