---
title: "Deep Solidity: What is the difference between private, internal, public and external functions?"
date: 2023-11-13T18:05:43-03:00
draft: true
---

**Q**: What is the difference between private, internal, public and external functions?

If you ask ChatGPT, it will tell you something like this:

```
In the context of Solidity for Ethereum smart contracts:

- Private: Accessible only within the contract they are defined in. Not even derived contracts can access them.
- Internal: Like private, but also accessible in derived contracts.
- Public: Accessible both internally and externally. Can be called by anyone.
- External: Only accessible from external calls. More gas-efficient for external calls compared to public functions. Not accessible internally within the contract (except using this).
```

This is all true, but a better way to show it is via a table. 

|           | Self  | Inherited         | External      |
| Private   |  YES  |      NO           |       NO      |
| Internal  |  YES  |     YES           |       NO      |
| Public    |  YES  |     YES           |       YES     |
| External  |  NO   |     NO            |       YES     |

ChatGPT also told us that `external` is for some reason more 
gas-efficient. 

At this point you should be asking:
1. **how** does Solidity prevent a `private` or `internal` methods from being called by external contracts?
2.  **why** are `external` methods more gas efficient? 

## Starting from the docs

The [Solidity docs on function visibility](https://docs.soliditylang.org/en/v0.8.23/contracts.html#function-visibility) just confirm what ChatGPT told us, but they introduce some interesting vocabulary:

- **Contract interface/ABI**:: the [docs](https://docs.soliditylang.org/en/v0.8.23/abi-spec.html#contract-abi-specification) have an specification for this; we only have to assume that they mean **Contract Application Binary Interface** when they say **Contract interface**, which to me sounds like a safe bet. 
- **Message calls**: [Solidity docs](https://docs.soliditylang.org/en/v0.8.23/introduction-to-smart-contracts.html#message-calls) covers us again. Message calls are what explorer know as `internal transactions`. I'm sad the `message call` terminology hasn't won but that is world we live in.

By reading the documentation one learns that the **Contract ABI** is `the standard way to interact with contracts in the Ethereum ecosystem` (note that this hints at the fact that there exist non-standard ways to interact with contracts). 

It mentions the **function selector** before getting into a lot of babble about argument encoding. The **function selector** is the part we care about: it tells us that `the first four bytes of the call data for a function call specifies the function to be called...`.

We can now answer question 1: how does solidity prevent `private` or `internal` methods? 

## Answering the first question

Well, if the **ABI** is the standard way to interact with contracts; and the documentation on visibility tells us the `private/internal` methods are _not_ part of the **ABI**, we have just found our answer. 

### Checking the source code

We can probably check if this is the case in the Solidity source code, right? I know next to nothing about compilers except that they like to use an [AST] (https://en.wikipedia.org/wiki/Abstract_syntax_tree), so after some digging I reach (`AsmParser.cpp::parseFunctionDefinition`)[https://github.com/ethereum/solidity/blob/14aed392614673505fe302fe1ac88bb351464217/libyul/AsmParser.cpp#L584]:

```cpp
FunctionDefinition Parser::parseFunctionDefinition()
{
	RecursionGuard recursionGuard(*this);

	if (m_currentForLoopComponent == ForLoopComponent::ForLoopPre)
		m_errorReporter.syntaxError(
			3441_error,
			currentLocation(),
			"Functions cannot be defined inside a for-loop init block."
		);

	ForLoopComponent outerForLoopComponent = m_currentForLoopComponent;
	m_currentForLoopComponent = ForLoopComponent::None;

	FunctionDefinition funDef = createWithLocation<FunctionDefinition>();
	expectToken(Token::Function);
	funDef.name = expectAsmIdentifier();
	expectToken(Token::LParen);
	while (currentToken() != Token::RParen)
	{
		funDef.parameters.emplace_back(parseTypedName());
		if (currentToken() == Token::RParen)
			break;
		expectToken(Token::Comma);
	}
	expectToken(Token::RParen);
	if (currentToken() == Token::RightArrow)
	{
		expectToken(Token::RightArrow);
		while (true)
		{
			funDef.returnVariables.emplace_back(parseTypedName());
			if (currentToken() == Token::LBrace)
				break;
			expectToken(Token::Comma);
		}
	}
	bool preInsideFunction = m_insideFunction;
	m_insideFunction = true;
	funDef.body = parseBlock();
	m_insideFunction = preInsideFunction;
	updateLocationEndFrom(funDef.debugData, nativeLocationOf(funDef.body));

	m_currentForLoopComponent = outerForLoopComponent;
	return funDef;
}
```

Read that. Isn't it weird? There no mention of a visibility? Also, `Token::RightArrow`? I must be misremembering my Solidity. What is even `FunctionDefinition`?

```cpp
/// Function definition ("function f(a, b) -> (d, e) { ... }")
struct FunctionDefinition { std::shared_ptr<DebugData const> debugData; YulString name; TypedNameList parameters; 
```

`function f(a,b) -> (d,e) { .. }`? That is **definitively** not what Solidity looks like. What gives? 

### Getting back on track

OK, first gotcha! There are two AST parsers in the Solidity project and we are looking at the wrong one, the one for [Yul](https://docs.soliditylang.org/en/latest/yul.html). Well that should be obvious we are in the `libyul` directory. Let's check out the `libsolidity` directory, and there it is, in [`Parser.cpp`](https://github.com/ethereum/solidity/blob/b0a986ffff0f077181e927d2e9f129c304b62828/libsolidity/parsing/Parser.cpp#L604-L605), this beauty:

(btw, yes, you can skip the snippet like all do, yes, i will metnion the important parts later)

```cpp
ASTPointer<ASTNode> Parser::parseFunctionDefinition(bool _freeFunction)
{
	RecursionGuard recursionGuard(*this);
	ASTNodeFactory nodeFactory(*this);
	ASTPointer<StructuredDocumentation> documentation = parseStructuredDocumentation();

	Token kind = m_scanner->currentToken();
	ASTPointer<ASTString> name;
	SourceLocation nameLocation;
	if (kind == Token::Function)
	{
		advance();
		if (
			m_scanner->currentToken() == Token::Constructor ||
			m_scanner->currentToken() == Token::Fallback ||
			m_scanner->currentToken() == Token::Receive
		)
		{
			std::string expected = std::map<Token, std::string>{
				{Token::Constructor, "constructor"},
				{Token::Fallback, "fallback function"},
				{Token::Receive, "receive function"},
			}.at(m_scanner->currentToken());
			nameLocation = currentLocation();
			name = std::make_shared<ASTString>(TokenTraits::toString(m_scanner->currentToken()));
			std::string message{
				"This function is named \"" + *name + "\" but is not the " + expected + " of the contract. "
				"If you intend this to be a " + expected + ", use \"" + *name + "(...) { ... }\" without "
				"the \"function\" keyword to define it."
			};
			if (m_scanner->currentToken() == Token::Constructor)
				parserError(3323_error, message);
			else
				parserWarning(3445_error, message);
			advance();
		}
		else
			tie(name, nameLocation) = expectIdentifierWithLocation();
	}
	else
	{
		solAssert(kind == Token::Constructor || kind == Token::Fallback || kind == Token::Receive, "");
		advance();
		name = std::make_shared<ASTString>();
	}

	FunctionHeaderParserResult header = parseFunctionHeader(false);

	ASTPointer<Block> block;
	nodeFactory.markEndPosition();
	if (m_scanner->currentToken() == Token::Semicolon)
		advance();
	else
	{
		block = parseBlock();
		nodeFactory.setEndPositionFromNode(block);
	}
	return nodeFactory.createNode<FunctionDefinition>(
		name,
		nameLocation,
		header.visibility,
		header.stateMutability,
		_freeFunction,
		kind,
		header.isVirtual,
		header.overrides,
		documentation,
		header.parameters,
		header.modifiers,
		header.returnParameters,
		block
	);
}
```

So this mentions ` FunctionHeaderParserResult header = parseFunctionHeader(false);`; we know the visibility is in the header so let's check it out:

```cpp
Parser::FunctionHeaderParserResult Parser::parseFunctionHeader(bool _isStateVariable) 
{
    ...

		else if (TokenTraits::isVisibilitySpecifier(token))
		{
			if (result.visibility != Visibility::Default)
			{
				// There is the special case of a public state variable of function type.
				// Detect this and return early.
				if (_isStateVariable && (result.visibility == Visibility::External || result.visibility == Visibility::Internal))
					break;
				parserError(
					9439_error,
					"Visibility already specified as \"" +
					Declaration::visibilityToString(result.visibility) +
					"\"."
				);
				advance();
			}
			else
				result.visibility = parseVisibilitySpecifier();
		}
}
```

OK, so there we go. We see that it parses the `visibility` of a function. The options aren't surprising, a function can be `public`, `internal`, `private` or `external`. 

### Codegen
So we've seen how Solidity will parse our method and assign a `visibility` to it. But what actions will it take depending on it? I had some hard time finding anything directly so I went to the Solidity folders again and found `libsolidity/codegen`. Yes, codegen, that's what we want. 

[`ContractCompiler::appendFunctionSelector`](https://github.com/ethereum/solidity/blob/2b8c9975b7118aaaf553f0db6e75377f48c36bd7/libsolidity/codegen/ContractCompiler.cpp#L411) seems promising. Let's see:

> Note that this is the old `codegen`, and modern Solidity will actually convert your code to Yul! Wanna read that? OK, [go here](https://github.com/ethereum/solidity/blob/2b8c9975b7118aaaf553f0db6e75377f48c36bd7/libsolidity/codegen/ir/IRGenerator.cpp#L317)

```cpp
void ContractCompiler::appendFunctionSelector(ContractDefinition const& _contract) {
    std::map<FixedHash<4>, FunctionTypePointer> interfaceFunctions = _contract.interfaceFunctions();

	std::map<FixedHash<4>, evmasm::AssemblyItem const> callDataUnpackerEntryPoints;

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

Well, it looks like that `interfaceFunctions` variable/getter is important. Let's see (`it`)[https://github.com/ethereum/solidity/blob/2a2a9d37ee69ca77ef530fe18524a3dc8b053104/libsolidity/ast/AST.cpp#L162-L163]:

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

> Nothing anything weird? The function is`interfaceFunctions(bool)`, but the caller passed no argument. 
> Don't worry, ccp is not broken. The default argument (specified in the header file) is `true`.

> BTW, notice that we just found the hash collision protection in solidity? Cool.

### Getting it all together

Ah, nice to see we are back in `AST.cpp`. AST feel comfy and familiar. Anyway, this `interfaceFunction` is kind of a lazy method that just calls `interfaceFunctionList`, so let us [check that out](https://github.com/ethereum/solidity/blob/2a2a9d37ee69ca77ef530fe18524a3dc8b053104/libsolidity/ast/AST.cpp#L274-L275).

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
			for (VariableDeclaration const* v: contract->stateVariables())
				if (v->isPartOfExternalInterface())
					functions.push_back(TypeProvider::function(*v));
			for (FunctionTypePointer const& fun: functions)
			{
				if (!fun->interfaceFunctionType())
					// Fails hopefully because we already registered the error
					continue;
				std::string functionSignature = fun->externalSignature();
				if (signaturesSeen.count(functionSignature) == 0)
				{
					signaturesSeen.insert(functionSignature);
					interfaceFunctionList.emplace_back(util::selectorFromSignatureH32(functionSignature), fun);
				}
			}
		}

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
> Nothing: it's an enum from most private to most public, so `external` is considered `bigger than` `public`.

`isOrdinary` is may be surprising, but some special functions like `construct` or `fallback` will return false here.

So we can finally be sure of something: Solidity will **not** put our function definitions in the ABI if they are not marked as `public` or `external`.

To recap, we checked:
1. That the AST will parse all methods and mark their visibility 
2. That `appendFunctionSelectors`, the code generator section in charge of deciding what functions will have function selectors or not, uses only those returned by the `interfaceFunctionList`.
3. That `interfaceFunctionList` only returns those marked as `public` or `external`

### It's not in the function dispatch, so what?

Mhm, but that's not enough to answer our question. Solidity will not put our function in the ABI, but how are we sure that only methods in the ABI are callable?

Well, that is a question Solidity actually cannot answer, and it has to do with `EVM` execution. You see, when calling a contract, the `EVM` assumes **absolutely nothing** and will just start executing contract code. The `EVM` has no knowledge of functions, types, or anything of the sort. 

To be sure of this, let's check `geth`, why not. Let use the latest release at the time of writing (`v1.13.4`) as our reference.

We will check out how a contract calls another contract (ie: a message call), but something similar happens when the code is executed via a transaction. The part of `geth` that deals with this is the (`evm::Call`)[https://github.com/ethereum/go-ethereum/blob/c39cbc1a78aa275523c1b0ff9d21b16ba7bfa486/core/vm/evm.go#L223-L224]:

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

Obviously the juiciy part is in (`evm.interpreter.Run`)[https://github.com/ethereum/go-ethereum/blob/fa6107c85e5717f10b2a57d470ead0e34b2152ba/core/vm/interpreter.go#L107]:

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

```
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

## And why are `external` methods more gas efficient?

Well, good thing we introduced EVM assembly just now, right? Because gas efficiency has more to do with the EVM and less to do with the solidity compiler (although the solidity compiler can help optimize the use!). 

In any case, starting from the end: `external` methods are more gas-efficient because they can read  from a sector of the memory call `CALLDATA`. 

### Yes, there are FIVE different memories

The `EVM` has different memories. If effect, it has five (!) distinct memory spaces:

- `STORAGE`, read/write, akin to a computer hard drive, for keeping data long term
- `MEMORY`, read-write memory akin to the `heap` on a normal computer, except it is not because it is not `random access` (not `RAM`), reading from higher positions **will** be expensive! 
- `STACK`, read/write and used only internally by the EVM, akin to a program's stack... except this one is totally disctict from the `heap`.
- `RETURNDATA`, writable by the callee and where the return data of a function will be 
- `CALLDATA`, read-only by the receiver of a CALL 

### So what?

So when you use `external` calls you are able to use the `CALLDATA` section of the memory to pass your arguments. 

And `CALLDATA` is _way_ cheaper than reading from memory:

```go
		CALLDATALOAD: {
			execute:     opCallDataLoad,
			constantGas: GasFastestStep, // This is 3
			minStack:    minStack(1, 1),
			maxStack:    maxStack(1, 1),
		},
```

```go
		MLOAD: {
			execute:     opMload,
			constantGas: GasFastestStep, // This is also 3
			dynamicGas:  gasMLoad,       // But has dynamic costs...
			minStack:    minStack(1, 1),
			maxStack:    maxStack(1, 1),
			memorySize:  memoryMLoad,
		},
```

### Well surely you can use `public` and perform `CALLs` when possible

No. You can't. See, Solidity needs to be prepared for your `public` method to be called internally. An in Solidity, "called internally" means performing a `JUMP`. And performing a `JUMP` means copying data to memory, not the calldata. 

If there is a chance that `CALL` will not be used, then the method must assume the worst and read from `memory` instead of `calldata`. 

To illustrate, let's use an example contract:

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.8.2 <0.9.0;

/**
 * @title Storage
 * @dev Store & retrieve value in a variable
 * @custom:dev-run-script ./scripts/deploy_with_ethers.ts
 */
contract Storage {
    function toHex2(uint256 a, uint256 b) public pure returns (uint256) {
            return a + b;
    }

    function toHex1(uint256 a, uint256 b) external pure returns (uint256) {
            return a + b;
    }
}
```

And let's analyze the decompilation of its runtime  to see what happened.

