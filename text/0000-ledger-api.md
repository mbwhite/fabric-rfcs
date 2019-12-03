---
layout: default
title: Ledger_API
parent: RFCs
---

- Feature Name: Ledger_API
- Start Date: 2019-12-03
- RFC PR: (leave this empty)
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

This feature will continue to extend the Fabric Programming Model, by clarifying the abstractions used to work with ledger data in a contract.
The scope of this is restricted to what is today referred to as the 'ChaincodeStub API'. We are focussing. 


# Motivation
[motivation]: #motivation

The motivation of this work is to provide a consistent set of concepts and APIs across all of the Contract languages and environments. Today that is Golang, and languages that run on the JVM and Node.js runtimes.

The updated Programming Model, has introduced higher level abstractions to both the contracts and the client SDKs. In many scenarios we have observed that users will write higher level classes to help them work with their business domain data. Not least the existing Fabric Samples, such as the Commercial Paper tutorial take this approach. 

This RFC (which is the first of a series) will start by formalizing the existing 'stub-api' into a conceptual architecture. A key principal in all the programming model work is that the conceptual model is consistent between languages. The polyglot developer will therefore have minimal friction moving to a different langauge.  Note that this a *conceptual* model; each implemetnation will use idomatic language idoms, and where practical, use the full expressive power of the language.

Haromnize the APIs ....

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Fabric documentation (v1.4 onwards) contains a chapter 'Developing Applications' which describes how to develop smart contracts and client applications using the new programming model. The chapter illustrates the concepts using the example of a commercial paper, and gives code examples as it guides the user through the development steps.  Though the code-examples are in JavaScript, the concepts explained there are applicable in Golang

A complete solution using Fabric can be represented as follows:
```
+-----------------------+      +-------------------------+       +-------------------------+
|                       |      |                         |       |                         |
| Application           |      | Smart Contract          |       | Ledger                  |
|                       |      |                         |       |                         |
| [organization-specific|      | [network-shared         |       | [network-shared         |
| application logic]    |      | transaction logic]      |       | transaction data]       |
|                       |      |                         |       |                         |
|                       |      |                         |       |                         |
|                       |      |                         |       |                         |
+--------+--------------+      +------------+------------+       +------------+------------+
         |                                  |                                 |
         |                                  |                                 |
         |                                  |                                 |
      +--+----------------------------------+---------------------------------+-------+
      |                                                                               |
      |  Network Channel          [network communication mechanism]                   |
      |                                                                               |
      |                                                                               |
      +-------------------------------------------------------------------------------+
```

A typical sequence would be for an application to connect to a peer (the 'gateway' into the network channel), and obtain a reference to a smart contract. Network-shared transaction logic would then be invoked by name, and with optional data arguments and return values. This logic will reference and update the shared transaction data. 

The 'Ledger' is, therefore, a primary abstraction in a blockchain system, including in Hyperledger Fabric; this abstraction allows the concepts of states, world state, collections, transactions, blocks etc to be tied together and formalised. 

## Ledger

A Ledger's current value is held in a set of collections. A single ledger is available to any given deployed contract, though other ledgers may be available within the Network Channel. 

Of the collections that are reference by the ledger, there a single collection in the ledger that is public (the 'WorldState') and held by every organization in the Network Channel. A ledger may have 1 or more private collections that are available only in some ogranizations.

## Collection

A collection is a set of states, with each state holding a business object or data. Each state being addressed by a key. Private collections are indentified by name, and held within a set of policy-defined ogranizations.

## State

A state holds the value of a business object or data, addressed by a key. The format of this object or data is defined by the overall solution. The direction of the programming model is to provide a direct ability to defined the structure of the this data. 

Access to individual states can be controlled by state-level endorsement policies.

As an example of how the concepts above can be reliased in code we can walkthough an updated version of the Java Commercial Paper example. 

The 'issue' method will create a new business object based on supplied data, and store this within the Ledger.

```
	/**
	 * Issue commercial paper
	 */
	@Transaction(submit = true)
	public CommercialPaper issue(CommercialPaperContext ctx, String issuer, String paperNumber, String issueDateTime,
			String maturityDateTime, int faceValue) {

		// create an instance of the paper
		CommercialPaper paper = CommercialPaper.createInstance(issuer, paperNumber, issueDateTime, maturityDateTime,
				faceValue, issuer, "");

		// Smart contract, rather than paper, moves paper into ISSUED state
		paper.setIssued();

		// Newly issued paper is owned by the issuer
		paper.setOwner(issuer);

		// want to put this into the public collection, aka 'world state'
		Collection collection = Ledger.getLedger(ctx).getCollection(Collection.WORLD);

		// put into the collection 
		String key = State.makeComposite(CommercialPaper.class.getName(), new String[] { paperNumber });
		collection.putState(key, CommercialPaper.serialize(paper));
		
    	return paper;
	}
```

Creation and setting properties on the Paper is domain specific code currently written. 
Access to the ledger is achieved via the `Ledger.getLedger(ctx)` - ctx being a reference to the transactional context this method/function is executing within. Access then to the 'world-state' or default collection is via `getCollection(...)`

A key is needed and the `State.makeComposite()` provides a helper function to create this key. `collection.putState(key,value)` stores the state. 

This is very similar to the current implementation in overall flow; the essential difference being the access to the collection. This permits a simpler approach to private data collections. For example if this paper was to be stored in a private collection the change that would be required is `getCollection("MyPrivateDataCollectionName")`  no other changes are needed. The collection abstracts away the different APIs.

Simiarly retrieval of state is as follows:

```
		// get the state
		String key = State.makeComposite(CommercialPaper.class.getName(), new String[] { paperNumber });
		State paperState = collection.getState(key);

        // use domain specific code to deserialize the data
   		CommercialPaper paper = CommercialPaper.deserialize(paperState.getValue());
```

## StateBased Endorsement

```
		// How to use StateBased Endorsement
		Principal p1 = new Principal("Org1", Role.ADMIN);
		Principal p2 = new Principal("Org2", Role.ADMIN);

		Principal p3 = new Principal("Org1", Role.CLIENT);
		Principal p4 = new Principal("Org2", Role.PEER);

		StateBasedEndorsement sbe1 = StateBasedEndorsement.build(
				Expression.and(
						Expression.or(p1,p2),
						Expression.or(p3,p4)
						)
				);
		paperState.setEndorsement(sbe1);
		
		
		// OR
		StateBasedEndorsement sbe2 = StateBasedEndorsement.build("AND( OR ('Org1.Admin','Org2.Admin) , OR('Org2.Client'',Org2.Admin'))");
		paperState.setEndorsement(sbe2);
		
```

## Query

```
		String startKey =  State.makeComposite(CommercialPaper.class.getName(), new String[] { startPaper }) ;
		String endKey =  State.makeComposite(CommercialPaper.class.getName(), new String[] { endPaper }) ;
		
		KeyQueryHandler keyQuery = KeyQueryHandler.RANGE;
		keyQuery.setFromKey( startKey ).setToKey(endKey);
		
		CollectionIterable<State> states = worldCollection.getStates(keyQuery);
		for (State s : states) {
			CommercialPaper paper = CommercialPaper.deserialize(s.getValue());
			totalValue += paper.getFaceValue();
		}
```


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

| Target Object                  | Function                             | Java Rendering                                                      | Existing Java Method                                                                                   |
| ------------------------------ | ------------------------------------ | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **Ledger**                     | get-collection-default             | Collection getDefaultCollection()                                 |                                                                                                        |
|                                | get-collection-private-name          | Collection getCollection(String name)                               |                                                                                                        |
|                                | bootstrap-ledger-api                 | Ledger Ledger.getLedger(ctx)                                        |                                                                                                        |
|                                |                                      |                                                                     |                                                                                                        |
| Collection                     | get-state                            | State getState(String  key)                                         | getState(String key);                                                                                  |
|                                |                                      |                                                                     | getPrivateData(String collection, String key);                                                         |
|                                |                                      |                                                                     |                                                                                                        |
|                                | put-state                            | State putState(String key, byte[] value)                            | putState(String key, byte[] value);                                                                    |
|                                |                                      |                                                                     |                                                                                                        |
|                                |                                      |                                                                     | putPrivateData(String collection, String key, byte[] value);                                           |
|                                |                                      |                                                                     |                                                                                                        |
|                                | delete-state                         | void deleteState(String key)                                        | delState(String key);                                                                                  |
|                                |                                      |                                                                     | delPrivateData(String collection, String key);                                                         |
|                                |                                      |                                                                     |                                                                                                        |
|                                | current-ledger-state-query           | Iterable<State> getStates(query Query)                              | getStateByRange(String startKey, String endKey);                                                       |
|                                |                                      |                                                                     | getStateByRangeWithPagination(String startKey, String endKey, int pageSize, String bookmark);          |
|                                |                                      |                                                                     | getStateByPartialCompositeKey(String compositeKey);                                                    |
| Query                          | query-by-key-range                   |                                                                     | getStateByPartialCompositeKey(String objectType, String... attributes);                                |
| (query handlers that determine | query-by-partial-key                 |                                                                     | getStateByPartialCompositeKey(CompositeKey compositeKey);                                              |
| how the query is done)         | query-from-key                       |                                                                     | getStateByPartialCompositeKeyWithPagination(CompositeKey compositeKey, int pageSize, String bookmark); |
|                                | query-to-key                         |                                                                     | getPrivateDataByRange(String collection, String startKey, String endKey);                              |
|                                | query-selector                       |                                                                     | getPrivateDataByPartialCompositeKey(String collection, String compositeKey);                           |
|                                |                                      |                                                                     | getPrivateDataByPartialCompositeKey(String collection, CompositeKey compositeKey);                     |
|                                |                                      |                                                                     | getPrivateDataByPartialCompositeKey(String collection, String objectType, String... attributes);       |
|                                |                                      |                                                                     | getQueryResult(String query);                                                                          |
|                                |                                      |                                                                     | getPrivateDataQueryResult(String collection, String query);                                            |
|                                |                                      |                                                                     | getQueryResultWithPagination(String query, int pageSize, String bookmark);                             |
|                                |                                      |                                                                     |                                                                                                        |
| StateIterator                  | next-state                           | Std java Collections Iteratables                                    |                                                                                                        |
|                                | has-next-state                       | Std java Collections Iteratables                                    |                                                                                                        |
|                                | close-iterable                       | Std java Collections Iteratables                                    |                                                                                                        |
|                                |                                      |                                                                     |                                                                                                        |
| HistoricStateIterator          | next-historic-state                  | Std java Collections Iteratables                                    |                                                                                                        |
|                                | has-next-historic-state              | Std java Collections Iteratables                                    |                                                                                                        |
|                                | close-iterable                       | Std java Collections Iteratables                                    |                                                                                                        |
|                                |                                      |                                                                     |                                                                                                        |
| State                          | create-composite-key                 | `String createCompositeKey(String objectType, String.. Attributes)` | createCompositeKey(String objectType, String... attributes);                                           |
|                                | split-composite-key                  | Array<String> getSplitKey(String compositeKey)                      | splitCompositeKey(String compositeKey);                                                                |
|                                | get-value                            | byte[] getValue()                                                   |                                                                                                        |
|                                | get-key                              | String getKey()                                                     |                                                                                                        |
|                                | get-key-level-endorsement            | byte[] getValidationParameter()                                     |                                                                                                        |
|                                |                                      |                                                                     |                                                                                                        |
|                                |                                      |                                                                     |                                                                                                        |
|                                | set-key-level-endorsement            | setValidationParameter(byte[])                                      |                                                                                                        |
|                                |                                      |                                                                     |                                                                                                        |
|                                |                                      |                                                                     |                                                                                                        |
|                                | get-private-data-hash                | byte[] getHash();                                                   |                                                                                                        |
|                                | get-policy (precise meaning TBD)     | byte[] getPolicy()                                                  |                                                                                                        |
|                                | get-historic-states-for-key          | Iterable<HistoricState> getHistory()                                | getHistoryForKey(String key);                                                                          |
|                                |                                      |                                                                     |                                                                                                        |
| StateHistory                   | get-value                            | byte[] getValue()                                                   |                                                                                                        |
|                                | get-tx                               | BasicTransaction getTransaction()                                   |                                                                                                        |
|                                | get-key                              | String getKey()                                                     |                                                                                                        |
|                                | get-is-delete                        | boolean isDelete()                                                  |                                                                                                        |
|                                |                                      |                                                                     |                                                                                                        |


# Drawbacks
[drawbacks]: #drawbacks

It could be argued that this is not needed because all of its capabilities can already be done in the existing apis.

This is being proposed not because of any underlying deficiencies in the existing APIs, but rather to provide a consistent set of concepts and structures across all of the supported developer APIs in Fabric.

# Rationale and alternatives
[alternatives]: #alternatives

This is the amalamation of experience in the field having seen solutions be implemented and the challenges encountered.
There are alternatives in how the APIs can be expressed, and there is always scope for variation.

# Prior art
[prior-art]: #prior-art

Significant experience and feedback has now been gained from user adoption of this programming model in Node and Java which have led to incremental improvements.

We know custom written code that as been used in different projects all of which has contributing to the thinking.

# Unresolved questions
[unresolved]: #unresolved-questions

None.