---
title: "Solidity Sceptics (1): The Difference between Private, Internal, Public and External Functions"
date: 2023-11-30T18:05:43-03:00
draft: false
---

> I intend this to be part of a series: "Solidity for Sceptics".
> There are [140 somewhat well-defined Solidity interview questions](https://www.rareskills.io/post/solidity-interview-questions). We will go over each of them in the most degenerate, unnecessary, depth; dispelling myths and always showing evidence for our claims.

In this first post, we will explore in depth the difference between private, internal, public and external functions. You probably think you know this topic well enough. Trust me, **you don't know it in the way you will know it after you finish reading this post**.

## Q1: What is the Difference Between Private, Internal, Public, and External Functions?

If you ask ChatGPT, you will be wrong. It will tell you something like this:G

```text
In the context of Solidity for Ethereum smart contracts:

- Private: Accessible only within the contract they are defined in. Not even derived contracts can access them.
- Internal: Like private, but also accessible in derived contracts.
- Public: Accessible both internally and externally. Can be called by anyone.
- External: Only accessible from external calls. More gas-efficient for external calls compared to public functions. Not accessible internally within the contract (except using this).
```

While this does represent the common wisdom, we will show that external methods _are_ accessible within the contract; although outside Solidity. They are also not any more or less gas efficient than `public` methods. Actually, we will see that `public` and `external` methods are pretty much exactly **the same**.

We will also explore the inner workings of `solc`, its [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) generator, we will use a bit of [Yul](https://docs.soliditylang.org/en/latest/yul.html) and we will see some code dealing with Solidity's inheritance.

Before we start, let's summarize the common wisdom in a table:

|           | Self  | Inherited         | External      |
|-----------|-------|-------------------|--------------|
| Private   |  YES  |      NO           |       NO      |
| Internal  |  YES  |     YES           |       NO      |
| Public    |  YES  |     YES           |       YES     |
| External  |  NO   |     NO            |       YES     |

Ready? Let us start our research from the most obvious place: the docs.

## Starting from the docs

The [Solidity docs on function visibility](https://docs.soliditylang.org/en/v0.8.23/contracts.html#function-visibility) seem to only confirm what ChatGPT told us, but they introduce some interesting vocabulary:

- **Contract interface/ABI**:: the [docs](https://docs.soliditylang.org/en/v0.8.23/abi-spec.html#contract-abi-specification) have an specification for this; we only have to assume that they mean **Contract Application Binary Interface** when they say **Contract interface**, which to me sounds like a safe bet.
- **Message calls**: [Solidity docs](https://docs.soliditylang.org/en/v0.8.23/introduction-to-smart-contracts.html#message-calls) covers us again. Message calls are what explorers sometimes know as `internal transactions`. I'm sad the `message call` terminology hasn't won but that is world we live in.

By reading the documentation you can learn that the **Contract ABI** is `the standard way to interact with contracts in the Ethereum ecosystem`. It mentions the **function selector** before getting into a lot of babble about argument encoding. The **function selector** is quite important: it is `the first four bytes of the call data for a function call specifies the function to be called...`.

This is as far as the documentations will get us. We learned that `public` and `external` are part of the **ABI**, while `private` and `internal` are not.

I don't think the docs quite work as evidence. While they are a nice starting point, let us go deeper to see how these visibility modifiers are actually implemented and how they behave in practice.

## Solidity's trees

So where should we start looking for details about the implementation of the visibility modifiers in `solc`?

I know little about compilers. Actually, one of the few things I know is that they use an [`AST`](<https://en.wikipedia.org/wiki/Abstract_syntax_tree>). An `AST` is just a tree holding different statements and expressions of a program. See, say my program is the simple `3 + (4 * 2)`. An possible AST for that is:

```text
    +
   / \
  3   *
     / \
    4   2
```

Anyway, because they are somewhat simple and _very_ useful, let us start our investigation of `solc`'s implementation of its visibility identifiers there.

Let's check out the `libsolidity` directory, and there it is, in [`Parser.cpp`](https://github.com/ethereum/solidity/blob/b0a986ffff0f077181e927d2e9f129c304b62828/libsolidity/parsing/Parser.cpp#L604-L605), this beauty:

```cpp
ASTPointer<ASTNode> Parser::parseFunctionDefinition(bool _freeFunction)
{
 ...
 Token kind = m_scanner->currentToken();
 ASTPointer<ASTString> name;
 SourceLocation nameLocation;
 ...

 FunctionHeaderParserResult header = parseFunctionHeader(false);
 ...
}
```

So this mentions `FunctionHeaderParserResult header = parseFunctionHeader(false);`. We know the visibility is in the header so let's check it out:

```cpp
Parser::FunctionHeaderParserResult Parser::parseFunctionHeader(bool _isStateVariable) 
{
  ...

  else if (TokenTraits::isVisibilitySpecifier(token))
  {
   if (result.visibility != Visibility::Default)
   {
     ...
   }
   else
    result.visibility = parseVisibilitySpecifier();
  }
}
```

OK, so there we go. We see that it parses the `visibility` of a function. The options aren't surprising, a function can be `public`, `internal`, `private` or `external`.

> **Research backstage:** I wasted a good while actually
looking into Yul's AST parsing before realizing I was in the wrong module. It is fun anyway, you can check out places like [`AsmParse.cpp::parseFunctionDefinition`](<https://github.com/ethereum/solidity/blob/14aed392614673505fe302fe1ac88bb351464217/libyul/AsmParser.cpp#L584>) if you do not have anything more fun to do today.

## Codegen

So we've seen how Solidity will parse our method and assign a `visibility` to it. But what actions will it take depending on it? I had some hard time finding anything useful from the AST module so I went to the Solidity folders again and found `libsolidity/codegen`. That seems like what we want.

[`ContractCompiler::appendFunctionSelector`](https://github.com/ethereum/solidity/blob/2b8c9975b7118aaaf553f0db6e75377f48c36bd7/libsolidity/codegen/ContractCompiler.cpp#L411) looks promising. Let's see:

> Note that this is the old `codegen`, and modern Solidity will actually convert your code to Yul! Wanna read that? OK, [go here](https://github.com/ethereum/solidity/blob/2b8c9975b7118aaaf553f0db6e75377f48c36bd7/libsolidity/codegen/ir/IRGenerator.cpp#L317).

```cpp
void ContractCompiler::appendFunctionSelector(ContractDefinition const& _contract) {
    std::map<FixedHash<4>, FunctionTypePointer> interfaceFunctions = _contract.interfaceFunctions();
    ...
 // retrieve the function signature hash from the calldata
 if (!interfaceFunctions.empty())
 {
  CompilerUtils(m_context).loadFromMemory(0, IntegerType(CompilerUtils::dataStartOffset * 8), true, false);

  // stack now is: <can-call-non-view-functions>? <funhash>
  std::vector<FixedHash<4>> sortedIDs;
  for (auto const& it: interfaceFunctions)
  {
   callDataUnpackerEntryPoints.emplace(it.first, m_context.newTag());
   sortedIDs.emplace_back(it.first);
  }
  std::sort(sortedIDs.begin(), sortedIDs.end());
  appendInternalSelector(callDataUnpackerEntryPoints, sortedIDs, notFound, m_optimiserSettings.expectedExecutionsPerDeployment);
 }
}
```

`C++` sure can get chaotic quick. But there seems to be one key line here, the one calling `appendInternalSelector`. We know that Solidity will add selectors for function on the ABI. What is more, that is all iterating over the `interfaceFunctions`! If we know how functions are added to the `interfaceFunctions` array, we would have found exactly the place where Solidity chooses which functions to the ABI.

Let's read that [`interfaceFunctions`](https://github.com/ethereum/solidity/blob/2a2a9d37ee69ca77ef530fe18524a3dc8b053104/libsolidity/ast/AST.cpp#L162-L163) method:

```cpp
std::map<util::FixedHash<4>, FunctionTypePointer> ContractDefinition::interfaceFunctions(bool _includeInheritedFunctions) const
{
 auto exportedFunctionList = interfaceFunctionList(_includeInheritedFunctions);

 std::map<util::FixedHash<4>, FunctionTypePointer> exportedFunctions;
 for (auto const& it: exportedFunctionList)
  exportedFunctions.insert(it);

 solAssert(
  exportedFunctionList.size() == exportedFunctions.size(),
  "Hash collision at Function Definition Hash calculation"
 );

 return exportedFunctions;
}
```

## Getting it all together

This `interfaceFunction` is kind of a lazy method that just calls `interfaceFunctionList`, so let us [check that out](https://github.com/ethereum/solidity/blob/2a2a9d37ee69ca77ef530fe18524a3dc8b053104/libsolidity/ast/AST.cpp#L274-L275):

```cpp
std::vector<std::pair<util::FixedHash<4>, FunctionTypePointer>> const& ContractDefinition::interfaceFunctionList(bool _includeInheritedFunctions) const
{
 return m_interfaceFunctionList[_includeInheritedFunctions].init([&]{
  std::set<std::string> signaturesSeen;
  std::vector<std::pair<util::FixedHash<4>, FunctionTypePointer>> interfaceFunctionList;

  for (ContractDefinition const* contract: annotation().linearizedBaseContracts)
  {
   if (_includeInheritedFunctions == false && contract != this)
    continue;
   std::vector<FunctionTypePointer> functions;
   for (FunctionDefinition const* f: contract->definedFunctions())
    if (f->isPartOfExternalInterface())
     functions.push_back(TypeProvider::function(*f, FunctionType::Kind::External));

   ...

  return interfaceFunctionList;
 });
}
```

Well, look at that! We finally have where a `visibility` method from our AST node is used: `isPartOfExternalInterface`:

```cpp
   for (FunctionDefinition const* f: contract->definedFunctions())
    if (f->isPartOfExternalInterface())
     functions.push_back(TypeProvider::function(*f, FunctionType::Kind::External));
```

`FunctionDefinition::isPartOfExternalInterface` is more or less what you expect:

```cpp
 bool isPartOfExternalInterface() const override { return isOrdinary() && isPublic(); }
 bool isPublic() const { return visibility() >= Visibility::Public; }
 bool isOrdinary() const { return m_kind == Token::Function; }
```

> You might get smart and see the visibility() >= Visibility::Public and think what's up with that.
> Nothing: it's an enum from most private to most public, so `external` is considered `bigger` than `public`.

`isOrdinary` may be surprising, but some special functions like `construct` or `fallback` will return false here.

So we can finally be sure of something: Solidity will **not** put our function definitions in the ABI if they are not marked as `public` or `external`.

Well that was a ride. To recap, we checked:

1. That the AST will parse all methods and mark their visibility
2. That `appendFunctionSelectors`, the code generator section in charge of deciding what functions will have function selectors or not, uses only those returned by the `interfaceFunctionList`.
3. That `interfaceFunctionList` only returns those marked as `public` or `external`

### Are we sure we can't call something that is not in the function selector list?

So we now can be fairly convinced that `public` and `external` will be shown on the public interface and `private` and `internal` functions will not.

Well, that is a question Solidity actually cannot answer, and it has to do with `EVM` execution. You see, when calling a contract, the `EVM` assumes **absolutely nothing** and will just start executing contract code. The `EVM` has no knowledge of functions, types, or anything of the sort.

To be sure of this, let's check `geth`, why not. Let use the latest release at the time of writing (`v1.13.4`) as our reference.

We will check out how a contract calls another contract (ie: a message call), but something similar happens when the code is executed via a transaction. The part of `geth` that deals with this is the [`evm::Call`](https://github.com/ethereum/go-ethereum/blob/c39cbc1a78aa275523c1b0ff9d21b16ba7bfa486/core/vm/evm.go#L223-L224):

```go
 } else {
  // Initialise a new contract and set the code that is to be used by the EVM.
  // The contract is a scoped environment for this execution context only.
  code := evm.StateDB.GetCode(addr)
  if len(code) == 0 {
   ret, err = nil, nil // gas is unchanged
  } else {
   addrCopy := addr
   // If the account has no code, we can abort here
   // The depth-check is already done, and precompiles handled above
   contract := NewContract(caller, AccountRef(addrCopy), value, gas)
   contract.SetCallCode(&addrCopy, evm.StateDB.GetCodeHash(addrCopy), code)
   ret, err = evm.interpreter.Run(contract, input, false)
   gas = contract.Gas
  }
 }
```

Obviously the juiciy part is in [`evm.interpreter.Run`](https://github.com/ethereum/go-ethereum/blob/fa6107c85e5717f10b2a57d470ead0e34b2152ba/core/vm/interpreter.go#L107):

```go
 // Get the operation from the jump table and validate the stack to ensure there are
  // enough stack items available to perform the operation.
  op = contract.GetOp(pc)
  operation := in.table[op]
  cost = operation.constantGas // For tracing
  // Validate stack
```

As you see, the only thing that `geth` cares about is the `pc` or `program counter`. This allows Solidity to generate that function dispatch that we digged up.

### Checking the function dispatch

It would be interesting at this point to actually see the function dispatch in EVM assembly, right? OK, let's do it. Let us use as an example a snippet from some research we did at [Coinspect](https:coinspect.com) for the [Tornado Cash Attack](https://github.com/coinspect/learn-evm-attacks/tree/master/test/Business_Logic/TornadoCash_Governance):

```text
    } else if (var0 == 0x9ae697bf) {
        // Dispatch table entry for lockedBalance(address)
        var1 = msg.value;
    
        if (var1) { revert(memory[0x00:0x00]); }
    
        var1 = 0x03b7;
        var2 = 0x063b;
        var3 = msg.data.length;
        var4 = 0x04;
        var2 = func_2982(var3, var4);
        var2 = func_063B(var2);
        goto label_03B7;
```

That is literally, piece by piece, what Solidity will generate a dispatch for a method of signature `lockedBalance(address)`. Note the `else if`. It is just a big `if` table matching with the 4-byte `function selector`! If your selector is not on that list, there is no place to `goto`, and your function **will not** get excecuted.

## What have we got evidence for so far?

Well, we already knew that `solc`'s `AST` parser processed methods and marked what visibility they had; and then the code generator uses this information to put `public`  and `external` methods on the dispatcher.

We now also saw the dispatcher and convinced ourselves that a `CALL` would always start executing from the dispatcher. If a method is not there, there is no way to get to it.

So we can answer in _extreme, excruciating_ detail the difference between `external/public` and `internal/private`.
These are two nice groups, but we need four. So let's now investigate the difference between `external`  and `public` and we will leave `internal` and `private` for the end.

### `external` vs `public` functions

We actually know their difference from the time (long ago) when we checked the Solidity docs: `external` methods are called _only_ via externally, via the `CALL` opcode (and its siblings `DELEGATECALL` and `STATICCALL`). `public` methods, on the other hand, can be called externally _or_ internally.

Checking this is the case its simple. Just try a test contract. You'll see it is right. But we want to go even deeper. _How_ is this difference actually implemented in the contract?

We will need to brush up our `assembly` and our `Yul` for this, so buckle up.

#### Solidity calling convention

To understand what the hell is happening here a primer
on Solidity's calling convention would be helpful. But I don't have will to go through it in depth here (this post is already longer than it should be). Luckily, the guys at [smlxl](https://smlxl.io/) have a quite recent Medium post on the topic, [so go read that](https://blog.smlxl.io/solc-internals-part-1-calling-conventions-89d62249ccda), probably.

Ready? OK. Let us move forward then. Honestly, all you should know is that JUMPs are used to go to functions.

#### Creating our test contract

Let us look at a simple contract that call a `public` method internally and compare it to a simple contract with an `external` method.

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity =0.8.23;

contract Foo {
    function foo(uint256 a) external returns (uint256) {
        return a + 0xC0FFEEBABE;
    }
}
```

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity =0.8.23;

contract Foo {
    function foo(uint256 a) public returns (uint256) {
        return a + 0xC0FFEEBABE;
    }
}
```

We can now do `solc --asm Contract.sol > Contract.asm` to see the difference in the assembly.

```diff
--- External.asm 2023-11-22 23:27:47.068056153 -0300
+++ Public.asm 2023-11-22 23:27:39.448184232 -0300
@@ -1,7 +1,7 @@

-======= External.sol:Foo =======
+======= Public.sol:Foo =======
 EVM assembly:
-    /* "External.sol":63:175  contract Foo {... */
+    /* "Public.sol":63:173  contract Foo {... */
   mstore(0x40, 0x80)
   callvalue
   dup1
@@ -23,7 +23,7 @@
 stop

 sub_0: assembly {
-        /* "External.sol":63:175  contract Foo {... */
+        /* "Public.sol":63:173  contract Foo {... */
       mstore(0x40, 0x80)
       callvalue
       dup1
@@ -46,7 +46,7 @@
       0x00
       dup1
       revert
-        /* "External.sol":82:173  function foo(uint256 a) external returns (uint256) {... */
+        /* "Public.sol":82:171  function foo(uint256 a) public returns (uint256) {... */
     tag_3:
       tag_4
       0x04
@@ -79,23 +79,23 @@
       swap1
       return
     tag_7:
-        /* "External.sol":124:131  uint256 */
+        /* "Public.sol":122:129  uint256 */
       0x00
-        /* "External.sol":154:166  0xC0FFEEBABE */
+        /* "Public.sol":152:164  0xC0FFEEBABE */
       0xc0ffeebabe
-        /* "External.sol":150:151  a */
+        /* "Public.sol":148:149  a */
       dup3
-        /* "External.sol":150:166  a + 0xC0FFEEBABE */
+        /* "Public.sol":148:164  a + 0xC0FFEEBABE */
       tag_11
       swap2
       swap1
       tag_12
       jump // in
     tag_11:
-        /* "External.sol":143:166  return a + 0xC0FFEEBABE */
+        /* "Public.sol":141:164  return a + 0xC0FFEEBABE */
       swap1
       pop
-        /* "External.sol":82:173  function foo(uint256 a) external returns (uint256) {... */
+        /* "Public.sol":82:171  function foo(uint256 a) public returns (uint256) {... */
       swap2
       swap1
       pop
@@ -357,6 +357,6 @@
       pop
       jump // out

-    auxdata: 0xa264697066735822122012f9e27c0a37151d2de3271a2ae4a227290c3244fc7b53f14632162835ccabc564736f6c63430008170033
+    auxdata: 0xa2646970667358221220f6aa36d71de4b64a4f18fe974c6c3f042d19f4d29e3abb5ccd86b8eed4dced1664736f6c63430008170033
 }
```

Surprise? There is no semantic difference, it is all comments and `auxdata` and whatnot.

This hints at the fact that there is **actually no difference between `public` and `external` functions in the bytecode**. The impossibility to call an `external` function from inside a contract is compiler-enforced: if we somehow get past the compiler, we could call an `external` method internally.

We can test this hypothesis by actually bypassing Solidity's compiler: we can use `yul` directly.

> **Research backstage:** I actually wanted to use Solidity's assembly blocks; but they don't allow you to use `JUMP`s, so dream was dead from the start.

First, let's see how a call to an `external` function _should_ look by creating a call to a `public` one (our hypthosesis is that they are the same, so they should be interchangeable).

We create a new contract, `PublicCalled.sol` that simply has a `caller()` method that calls a `public` function:

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity =0.8.23;

contract Foo {
    function foo(uint256 a) public returns (uint256) {
        return a + 0xC0FFEEBABE;
    }

    function caller() external returns (uint256) {
        return foo(0xAABBCCDD);
    }
}
```

And we are intersted in the `yul` version of this contract, so we use `solc --ir` to get both the original `Public.sol` (without the `caller()`) and `PublicCalled.sol`.

```bash
solc --ir Public.sol > Public.yul && solc --ir PublicCalled.sol > PublicCalled.yul
```

If we `diff` them we can quickly find two big differences introduced in `PublicCalled.yul`:

```diff
+            function external_fun_caller_23() {
+
+                if callvalue() { revert_error_ca66f745a3ce8ff40e2ccaf1ad45db7774001b90d25810abd9040049be7bf4bb() }
+                abi_decode_tuple_(4, calldatasize())
+                let ret_0 :=  fun_caller_23()
+                let memPos := allocate_unbounded()
+                let memEnd := abi_encode_tuple_t_uint256__to_t_uint256__fromStack(memPos , ret_0)
+                return(memPos, sub(memEnd, memPos))
+
+            }

+            function fun_caller_23() -> var__16 {
+                /// @src 0:213:220  "uint256"
+                let zero_t_uint256_3 := zero_value_for_split_t_uint256()
+                var__16 := zero_t_uint256_3
+
+                /// @src 0:243:253  "0xAABBCCDD"
+                let expr_19 := 0xaabbccdd
+                /// @src 0:239:254  "foo(0xAABBCCDD)"
+                let _4 := convert_t_rational_2864434397_by_1_to_t_uint256(expr_19)
+                let expr_20 := fun_foo_13(_4)
+                /// @src 0:232:254  "return foo(0xAABBCCDD)"
+                var__16 := expr_20
+                leave
+
+            }
```

It looks like the first section represents some simple sanity checks (no value call, because the method is not payable) and the return logic; while `fun_caller_23() -> var_16` is more interesting for us, actually calling the public method:

```text
let expr_20 := fun_foo_13(_4);
```

Let's see if we can hack together a call to an `external` function using this exact same logic! Time to get our hands dirty. Want to know the recipe?

1. Get a `yul` version of `External.sol`: `solc --ir External.sol > External.yul`.
2. Copy the diff of `PublicCaller.yul` with `Public.yul`
3. Paste the diff into `External.yul`. Mind the places where you copy!
4. Name the new file `ExternalHacked.yul`

Voi la. Let's diff our `External.yul` with our `ExternalHacked.yul` just to make sure it somewhat makes sense:

```diff
--- External.yul 2023-11-23 00:11:25.882824027 -0300
+++ ExternalHacked.yul 2023-11-23 00:00:23.680475303 -0300
@@ -1,5 +1,3 @@
-IR:
-
 /// @use-src 0:"External.sol"
 object "Foo_14" {
     code {
@@ -49,8 +47,31 @@
                     external_fun_foo_13()
                 }

+                case 0xfc9c8d39
+                {
+                    // caller()
+
+                    external_fun_caller_23()
+                }
+
+
                 default {}
             }
+            function abi_decode_tuple_(headStart, dataEnd)   {
+                if slt(sub(dataEnd, headStart), 0) { revert_error_dbdddcbe895c83990c08b3492a0e83918d802a52331272ac6fdb6a7c4aea3b1b() }
+
+            }
+
+            function external_fun_caller_23() {
+
+                if callvalue() { revert_error_ca66f745a3ce8ff40e2ccaf1ad45db7774001b90d25810abd9040049be7bf4bb() }
+                abi_decode_tuple_(4, calldatasize())
+                let ret_0 :=  fun_caller_23()
+                let memPos := allocate_unbounded()
+                let memEnd := abi_encode_tuple_t_uint256__to_t_uint256__fromStack(memPos , ret_0)
+                return(memPos, sub(memEnd, memPos))
+
+            }

             revert_error_42b3090547df1d2001c96683413b8cf91c1b902ef5e3cb8d9f6f304cf7446f74()

@@ -179,6 +200,32 @@
                 leave

             }
+
+            function cleanup_t_rational_2864434397_by_1(value) -> cleaned {
+                cleaned := value
+            }
+
+            function convert_t_rational_2864434397_by_1_to_t_uint256(value) -> converted {
+                converted := cleanup_t_uint256(identity(cleanup_t_rational_2864434397_by_1(value)))
+            }
+
+            /// @ast-id 23
+            /// @src 0:177:261  "function caller() external returns (uint256) {..."
+            function fun_caller_23() -> var__16 {
+                /// @src 0:213:220  "uint256"
+                let zero_t_uint256_3 := zero_value_for_split_t_uint256()
+                var__16 := zero_t_uint256_3
+
+                /// @src 0:243:253  "0xAABBCCDD"
+                let expr_19 := 0xaabbccdd
+                /// @src 0:239:254  "foo(0xAABBCCDD)"
+                let _4 := convert_t_rational_2864434397_by_1_to_t_uint256(expr_19)
+                let expr_20 := fun_foo_13(_4)
+                /// @src 0:232:254  "return foo(0xAABBCCDD)"
+                var__16 := expr_20
+                leave
+
+            }
             /// @src 0:63:175  "contract Foo {..."

         }
```

That looks reasonable. The `IR:` at the top that was removed is just an artifact and `solc` complains when trying to compile the `yul` if it is present. Everything else is an addition, and is the _exact same code_ that was added to `Public.sol` when we introduced the call via the `caller()` method.

We now have our test almost ready. We just need to deploy it and see if it works.

 To get the bytecode for us to deploy, we run `solc --strict-assembly ExternalHacked.yul` and we look for the `Binary Representation` section:

```text
Binary representation:
60806040523461001f57610011610024565b61022c61002f823961022c90f35b61002a565b60405190565b5f80fdfe60806040526004361015610013575b61012f565b61001d5f35610080565b80632fbebd38146100375763fc9c8d390361000e5761004b565b6100fa565b5f91031261004657565b610090565b3461007b5761005b36600461003c565b6100776100666101d5565b61006e610086565b918291826100e5565b0390f35b61008c565b60e01c90565b60405190565b5f80fd5b5f80fd5b90565b6100a081610094565b036100a757565b5f80fd5b905035906100b882610097565b565b906020828203126100d3576100d0915f016100ab565b90565b610090565b6100e190610094565b9052565b91906100f8905f602085019401906100d8565b565b3461012a576101266101156101103660046100ba565b610192565b61011d610086565b918291826100e5565b0390f35b61008c565b5f80fd5b5f90565b90565b90565b61015161014c61015692610137565b61013a565b610094565b90565b634e487b7160e01b5f52601160045260245ffd5b61017c61018291939293610094565b92610094565b820180921161018d57565b610159565b6101b39061019e610133565b506101ad64c0ffeebabe61013d565b9061016d565b90565b90565b6101cd6101c86101d2926101b6565b61013a565b610094565b90565b6101dd610133565b506101f36101ee63aabbccdd6101b9565b610192565b9056fea2646970667358221220d087827f599e26c3fbe86c81962f849a7f3fd16eadc4ae85bf500608d6fb415564736f6c63430008170033``
```

I used `Remix` to deploy it with the help of this [nice snippet-contract](https://github.com/Sekin/ethereum-bytecode-deployment/blob/master/DeployBytecode.sol) which allow us to deploy arbitrary bytecode. My bytecode was deployed at address `0x5C9eb5D6a6C2c1B3EFc52255C0b356f116f6f66D`.

So now I can just use Remix's JS console to call the `caller()` method (which has the signature `0xfc9c8d39`):

```js
web3.eth.call({to: "0x5C9eb5D6a6C2c1B3EFc52255C0b356f116f6f66D", data: "0xfc9c8d39"})
```

Lo and behold, the result is:

```text
0x000000000000000000000000000000000000000000000000000000c1aaaa879b
```

If you remember our contract, `caller` did `foo(0xAABBCCDD)` and `foo` just added its input with `0xC0FFEEBABE`. And `0xC0FFEEBABE + 0xAABBCCDD = 0xc1aaaa879b`!

There we go. We just called a Solidity's `external` method from inside the same contract without using `CALL`. Very practical, right?

This goes to show us that the difference between `external` and `public` methods are minimal, if existing at all. It is mostly a compiler-enforced invariant intended to clarify for programmers and other readers of the code in what context a function is _intended_ to be used. It has little to no bearing in the actual deployed bytecode.

> BTW, **Solidity is not broken**. You _cannot_ call an `external` method from inside the same contract, _as long as you stay inside Solidity_. Once you move to `yul`, another whole set of invariants are on the table. Nevertheless, it was interesting to confirm that `external` and `public` functions are only different from Solidity's point of view and not from the `EVM`'s point of view.

## What a ride. So what can we say we know now?

We _already knew_ that:

- Visibilities are parsed by the AST
- Code is generated by the `codegen` module
- `codegen` puts the functions in the dispatcher only if they are `public` or `external`
- `CALL`s always start from the dispatcher

And we now also know that:

- `external` and `public` differences exist only inside Solidity
- `external` and `public` functions look the the same from the `EVM`'s point of view
- You **can** call an `external` method from the same contract, you just have to bypass solidity.

I guess the only thing now to research are the differences between `internal` and `private` functions. This will need at least an introduction to Solidity's inheritance and is the part I'm most dreading researching, so let's be like Courage the Cowardly Dog and keep on going.

![courage the dog as a hacker](courage-dog.png)

## Internal vs Private

Recall our common wisdom, which said that the only difference
between `internal` and `private` methods is that `internal` can be called from inherited contracts and `private` are exclusive
to the contract where they are defined.

The obvious test gives us the expected result:

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity =0.8.23;

contract Parent {
    function parent(uint256 a) private returns (uint256) {
        return a + 0xC0FFEEBABE;
    }
}

contract Child is Parent {

    function child() external {
        parent(0xC0FFEEBABE); <-- ERROR: Undeclared identifier
    }

}
```

We got on a good track last time by reviewing a minimal contract and looking at the bytecode, so let's repeat that.

Let's see if the bytecode between a simple private and internal method differs. First, for reference, these are the `.sol`:

```bash
$ cat Internal.sol Private.sol
// SPDX-License-Identifier: GPL-3.0

pragma solidity =0.8.23;

contract Parent {
    function parent(uint256 a) internal returns (uint256) {
        return a + 0xC0FFEEBABE;
    }

    function caller() external {
        parent(0xAAAAAA);
    }
}
// SPDX-License-Identifier: GPL-3.0

pragma solidity =0.8.23;

contract Parent {
    function parent(uint256 a) private returns (uint256) {
        return a + 0xC0FFEEBABE;
    }

    function caller() external {
        parent(0xAAAAAA);
    }
}%                                  
```

> I had to include the `caller()` method in there because
if you don't, solidity will optimize away uncalled functions.

You would be not be surprised at this point if I told you that the `.yul`s you get with `solc --ir` differ in nothing but the comments.

But maybe Solidity was doing the same thing it does when we just don't call a method? Maybe I could see the difference only if I actually called it from an inherited contract? Let's try another minimal contract.

```bash
$ cat Internal.sol 
// SPDX-License-Identifier: GPL-3.0

pragma solidity =0.8.23;

contract Parent {
    function parent(uint256 a) internal returns (uint256) {
        return a + 0xC0FFEEBABE;
    }

    function caller() external {
        parent(0xAAAAAA);
    }
}

contract Child is Parent {
    function caller_child() external {
        parent(0xBBBBBB);
    }
}                     
```

When we analyze the Yul, we can see two very the sections corresponding to both `caller()` and `caller_child()`:

```text
            function fun_caller_child_32() {
                /// @src 0:330:338  "0xBBBBBB"
                let expr_28 := 0xbbbbbb
                /// @src 0:323:339  "parent(0xBBBBBB)"
                let _1 := convert_t_rational_12303291_by_1_to_t_uint256(expr_28)
                let expr_29 := fun_parent_13(_1)
            }

            function fun_caller_21() {
                /// @src 0:229:237  "0xAAAAAA"
                let expr_17 := 0xaaaaaa
                /// @src 0:222:238  "parent(0xAAAAAA)"
                let _2 := convert_t_rational_11184810_by_1_to_t_uint256(expr_17)
                let expr_18 := fun_parent_13(_2)
            }
```

Do you see a difference? Well, obviously one uses `0xbbbbbb` and the other one `0xaaaaaa` as the parameter to pass to `fun_parent_13()`. But otherwise, they are identical!

And how does the call look when made to an `private` method? We actually already now (remember, the diff we did came up empty except for comments), but let's check it out anyway.

This is the call to an `private` method as seen in our `Private.sol` file:

```text
            function fun_caller_21() {
                /// @src 0:228:236  "0xAAAAAA"
                let expr_17 := 0xaaaaaa
                /// @src 0:221:237  "parent(0xAAAAAA)"
                let _1 := convert_t_rational_11184810_by_1_to_t_uint256(expr_17)
                let expr_18 := fun_parent_13(_1)
            }
```

It is **exactly** the same call! So again we can conclude that, just like with `external` and `public` methods, the differences are only at the compiler level. There's nothing on the bytecode.

> Well the inheritance part was not that difficult after all <3

## All is protected by the same strategy

There is something in common when calling a `private` method from an inherited contract and an `external` method from the same contract: the error is always the same:

```text
Undeclared identifier. Did you mean... ?
```

Well... what better way to wrap this post up if not by looking for this string in `solc` and trying to understand what we see?

Just with a quick grep we reach the part of the code that emmits this error:

```cpp
bool ReferencesResolver::visit(Identifier const& _identifier)
{
 auto declarations = m_resolver.nameFromCurrentScope(_identifier.name());
 if (declarations.empty())
 {
  std::string suggestions = m_resolver.similarNameSuggestions(_identifier.name());
  std::string errorMessage = "Undeclared identifier.";
  ...
 }
```

If we follow the `m_resolver`, we will quickly see that it uses a member `m_declarations` which are registered via the `DeclarationContainer::registerDeclaration` method. The end
of that method gives us quite a lot of information:

```cpp
 std::vector<Declaration const*>& decls = _invisible ? m_invisibleDeclarations[*_name] : m_declarations[*_name];
 if (!util::contains(decls, &_declaration))
  decls.push_back(&_declaration);
 return true;
```

The `_invisible` flag decides if it is put into the `m_invisibleDeclarations` or the `m_declarations`.

> Solidity devs have left us a comments as to what
> the misterious invisible declarations are:

```cpp
 // We use "invisible" for both inactive variables in blocks and for members invisible in contracts.
 // They cannot both be true at the same time.
 solAssert(!(_inactive && !_declaration.isVisibleInContract()), "");
```

But the important part is this one, where it actually registers a declaration.

```cpp
 if (!_container.registerDeclaration(_declaration, _name, _errorLocation, !_declaration.isVisibleInContract() || _inactive, false))
```

See the call to `isVisibileInContract()`, we had seen that in our first research on `external` vs `public`!

```cpp
 virtual bool isVisibleInContract() const { return visibility() != Visibility::External; }
```

Soo if the method is `external`, it will not be registered. But we know we are missing something: derived contracts. Luckily, just below `isVisibleInContract()`, we have the ominous `isVisibleInDerivedContracts`,  which returns true only if it is `visibleInContract` (ie: not `external`) and NOT `private`.

Let's `grep` for where `isVisibleInDerivedContracts` is used and see if we find anything useful... bingo:

```cpp
void NameAndTypeResolver::importInheritedScope(ContractDefinition const& _base)
{
 auto iterator = m_scopes.find(&_base);
 solAssert(iterator != end(m_scopes), "");
 for (auto const& nameAndDeclaration: iterator->second->declarations())
  for (auto const& declaration: nameAndDeclaration.second)
   // Import if it was declared in the base, is not the constructor and is visible in derived classes
   if (declaration->scope() == &_base && declaration->isVisibleInDerivedContracts())
    if (!m_currentScope->registerDeclaration(*declaration, false, false))
    {
```

So Solidity will inherit a scope and `registerDeclaration` for all of the declarations that are `visibleInDerivedContracts`!

## Wrapping up

So, to reiterate, we had already seen how:

- Visibilities are parsed by the AST
- Code is generated by the `codegen` module
- `codegen` puts the functions in the dispatcher only if they are `public` or `external`
- `CALL`s always start from the dispatcher
- `external` and `public` differences exist only inside Solidity
- `external` and `public` functions look the the same from the `EVM`'s point of view
- You **can** call an `external` method from the same contract, you just have to bypass solidity.

And we now we've also seen that:

- The same story is going on for `private` and `internal`
- Solidity uses declarations to stop you from using methods were you should not use them
- These are all compiler-enforced protections and have no bearing on the contract bytecode

## End of part 1

That concludes our exploration. We've presented quite a lot of evidence for our claims, and now it's time to wrap up this post.

If you liked the post, please let me know! Writing this was a significant time commitment and specially if I intend to make it a series I need to know if someone is reading.

If you made it so far, remember to [follow me on twitter](https://twitter.com/0xmatebabe) and either DM or just reply with some feedback or your thoughts! These are complex topics and it is likely there are mistakes on this post, somewhere. I would like to correct them with time. If you spot any, let me know, I will of course provide appropriate credits.

In the next series, I intend to explore the second question on RareSkill's list: `Approximately, how large can a smart contract be?`. If expect that question to be significantly easier than the one dealing with visibilities, so if you were overwhelmed come back for that one!
