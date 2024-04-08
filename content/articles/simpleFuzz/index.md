# `Simple Fuzzing with Foundry and Echidna`

In this article we are going to use [Foundry](https://book.getfoundry.sh/getting-started/installation) and [Echidna](https://github.com/crytic/echidna) to break a simple contract. We are going to need [Docker](https://docs.docker.com/get-docker/) installed in order to use Echidna.

It will also be helpful if you are familiar with Foundry and its directory structure. You can find all the code that we will be using [here](https://github.com/wildanvin/simpleFuzz-foundry-echidna).

This is the contract that we will begin with:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract SimpleFuzz {

    uint256 public shouldAlwaysBeZero = 0;
    uint256 private hiddenValue = 0;

    function doStuff (uint256 data) public {
        if (data == 1234){
            shouldAlwaysBeZero = 1;
        }
    }
}
```

The `shouldAlwaysBeZero` variable, well, should always be zero at all costs. This will be our invariant, a property in our contract that should always be true.

A simple unit test with Foundry will look like this:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {SimpleFuzz} from "../src/SimpleFuzz.sol";

contract FoundrySimpleFuzz is Test {
    SimpleFuzz public simpleFuzz;

    function setUp() public {
        simpleFuzz = new SimpleFuzz();
    }

    function testSimpleDoStuff() public {
        simpleFuzz.doStuff(123);
        assert(simpleFuzz.shouldAlwaysBeZero() == 0);
    }
}
```

If we run `forge test --mt testSimpleDoStuff` the test will pass and won’t catch the bug. In this simple example is easy to see how to change the `shouldAlwaysBeZero` variable to 1, but we want a way to automatically detect it, because real world protocols are not that simple.

## Stateless fuzzing

With stateless fuzzing, our tools will make calls to the contract with random inputs in an attempt to break the invariant.

### 1. Foundry

To add stateless fuzzing in Foundry, we only need to add this function to our test contract:

```jsx
function testFuzzDoStuff(uint256 x) public {
        simpleFuzz.doStuff(x);
        assert(simpleFuzz.shouldAlwaysBeZero() == 0);
    }
```

Simply by adding a parameter to the function and an assertion Foundry will fuzz the parameter and try to break the invariant. Now if we run `forge test --mt testFuzzDoStuff -vvvvv` we will get:

```jsx
Failing tests:
Encountered 1 failing test in test/FoundrySimpleFuzz.t.sol:FoundrySimpleFuzz
[FAIL. Reason: Assertion violated Counterexample: calldata=0xe41f930100000000000000000000000000000000000000000000000000000000000004d2, args=[1234]] testFuzzDoStuff(uint256) (runs: 140, μ: 8319, ~: 8319)
```

If foundry didn’t find the edge case, try to increase the number of runs in your `foundry.toml` file:

```jsx
;[profile.default]
src = 'src'
out = 'out'
libs = ['lib'][fuzz]
runs = 5000
seed = '0x38'
```

### 2. Echidna

To use Echidna we need to create a new contract in the `test` folder:

```jsx
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {SimpleFuzz} from "../src/SimpleFuzz.sol";

contract EchidnaSimpleFuzz is SimpleFuzz {

    function echidna_invariant () public view returns (bool) {
        return (shouldAlwaysBeZero == 0);
    }

}
```

Very simple contract right?

Now to use Echidna run the following command:

```jsx
docker run -it --rm -v $PWD:/home/ethsec/code ghcr.io/trailofbits/eth-security-toolbox:nightly
```

If you haven’t used docker before (like me) that last command will look like gibberish, but fear not, here is chatGPT with a nice explanation of what it does:

- chatGPT explanation of docker command
  This command is used to run a Docker container, specifically one based on the `ghcr.io/trailofbits/eth-security-toolbox:nightly` image. Docker is a platform for developing, shipping, and running applications in containers, which are lightweight, standalone, and executable software packages that include everything needed to run a piece of software, including the code, runtime, system tools, libraries, and settings. Let's break down the command piece by piece:
  - `docker run`: This is the command used to run a new container. It tells Docker to pull the image if it's not already locally available and start a new container based on that image.
  - `it`: This option is actually two options combined. `i` stands for interactive, keeping the STDIN (standard input) open even if not attached. `t` allocates a pseudo-TTY, which means it simulates a terminal, like what you would get when you open a terminal emulator. Together, `it` makes it possible to interact with the container via the command line.
  - `-rm`: This option automatically removes the container when it exits. Containers can consume disk space, and removing them when you're done helps keep your system clean.
  - `v $PWD:/home/ethsec/code`: This is a volume mount option. `v` mounts a directory from your host into the container. `$PWD` is a variable in your shell that stands for "Print Working Directory," which is the current directory you're in on your host system. `:/home/ethsec/code` specifies the path inside the container where the host directory is mounted. This allows you to share files between your host system and the container. In this case, whatever is in the current directory on the host system will appear in `/home/ethsec/code` inside the container.
  - `ghcr.io/trailofbits/eth-security-toolbox:nightly`: This specifies the Docker image to use. `ghcr.io` is the GitHub Container Registry, a service for hosting container images. `trailofbits/eth-security-toolbox` is the name of the repository on GHCR, and `nightly` is the tag for the image, indicating this image is a nightly build, which is usually the latest development version of the software.
  In summary, this command runs a container interactively, with the current directory on the host system mounted into the container. The container is based on a nightly build of the Trail of Bits Ethereum Security Toolbox image. Once the container's process exits, the container itself is automatically removed to not leave any unnecessary clutter on your system. This setup is particularly useful for security analysis or development work related to Ethereum, as it provides a pre-configured environment with tools and libraries tailored for this purpose.

Once you are inside the docker image, switch to the `code` directory with `cd code` and run:

```jsx
echidna test/EchidnaSimpleFuzz.t.sol --contract EchidnaSimpleFuzz --test-limit 500
```

If everything is fine, Echidna will show you a nice screen with the case that breaks the invariant:

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/2b50c96a-2efe-4814-9663-16137417799d/6de6074e-3a23-4bd1-a601-c54ae2a04aa6/Untitled.png)

## Stateful fuzzing

Lets modify a bit our `SimpleFuzz.sol` contract:

```jsx
function doStuff (uint256 data) public {
        // if (data == 1234){
        //     shouldAlwaysBeZero = 1;
        // }
        if (hiddenValue == 5678) {
            shouldAlwaysBeZero = 1;
        }
        hiddenValue = data;
    }
```

Now it is a bit tricky to brake the invariant because you have to call the `doStuff` function two times. The first one to set the `hiddenValue` to 5678 and the second one that will set the `shouldAlwaysBeZero` variable to 1 (once `hiddenValue` is 5678). In other words we need to preserve the state of the first call to break the invariant

### 1. Foundry

No matter how many `runs` you add to your `foundry.toml` file, it won’t catch the bug. We need to add a couple of lines to our previous foundry test contract

```jsx
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {StdInvariant} from "forge-std/StdInvariant.sol";
import {SimpleFuzz} from "../src/SimpleFuzz.sol";

contract FoundrySimpleFuzz is StdInvariant, Test {
    SimpleFuzz public simpleFuzz;

    function setUp() public {
        simpleFuzz = new SimpleFuzz();
        targetContract(address(simpleFuzz));
    }

    function testSimpleDoStuff() public {
        simpleFuzz.doStuff(123);
        assert(simpleFuzz.shouldAlwaysBeZero() == 0);
    }

    // Stateless fuzzing
    function testFuzzDoStuff(uint256 x) public {
        simpleFuzz.doStuff(x);
        assert(simpleFuzz.shouldAlwaysBeZero() == 0);
    }

    //Stateful fuzzing aka invariant test:
    function invariant_testAlwaysReturnsZero () public view {
        assert(simpleFuzz.shouldAlwaysBeZero() == 0);
    }
}
```

Basically, our test needs to inherit from the `StdInvariant.sol` library, specify the contract we want to do stateful fuzzing with `targetContract` and finally write a function that test our invariant. Now if we run:

```jsx
forge test --mt invariant_testAlwaysReturnsZero
```

Foundry will give you a sequence of function calls that will get the `shouldAlwaysBeZero` to a state different than 0.

### 2. Echidna

Echidna is smarter in this case and it doesn’t need any more configuration. So if you start the docker image with:

```jsx
docker run -it --rm -v $PWD:/home/ethsec/code ghcr.io/trailofbits/eth-security-toolbox:nightly
```

Execute `cd code` and run:

```jsx
echidna test/EchidnaSimpleFuzz.t.sol --contract EchidnaSimpleFuzz --test-limit 500
```

Equidna will get the sequence call that breaks the invariant:

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/2b50c96a-2efe-4814-9663-16137417799d/2d5bca0f-b7de-41e3-a2a5-03e1181df8cb/Untitled.png)

## Conclusion

We covered the basics to use fuzzing with Foundry and Echidna. Check out other introductory resources that I borrowed heavily to write this article:

- Patrick’s Intro to Fuzzing [video](https://www.youtube.com/watch?v=juyY-CTolac&t=379s)
- Smart Contract Programmer [video](https://www.youtube.com/watch?v=vCTnI2nDnAw) on Echidna
- Solidity By Example on [Echidna](https://solidity-by-example.org/tests/echidna/)

Also, feel free to reach out on [twitter](https://twitter.com/wildanvin) if you have any questions or feedback **✌️**
