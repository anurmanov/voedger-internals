---
reqmd.package: server.design.sequences
---

# Sequences

## Motivation

As of March 1, 2025, the sequence implementation has several critical limitations that impact system performance and scalability:

- **Unbound Memory Growth**: Sequence data for all workspaces is loaded into memory simultaneously, creating a direct correlation between memory usage and the number of workspaces. This approach becomes unsustainable as applications scale.

- **Prolonged Startup Times**: During command processor initialization, a resource-intensive "recovery process" must read and process the entire PLog to determine the last used sequence numbers. This causes significant startup delays that worsen as event volume grows.

The proposed redesign addresses these issues through intelligent caching, background updates, and optimized storage mechanisms that maintain sequence integrity while dramatically improving resource utilization and responsiveness.

## Introduction

This document outlines the design for sequence number management within the Voedger platform.

A **Sequence** in Voedger is defined as a monotonically increasing series of numbers. The platform provides a unified mechanism for sequence generation that ensures reliable, ordered number production.

As of March 1, 2025, Voedger implements four specific sequence types using this mechanism:

- **PLogOffsetSequence**: Tracks write positions in the PLog
  - Starts from 1
- **WLogOffsetSequence**: Manages offsets in the WLog
  - Starts from 1
  - To read all events by SELECT
- **CRecordIDSequence**: Generates unique identifiers for CRecords
  - Starts from 322685000131072
    - [https://github.com/voedger/voedger/blob/ae787d43bf313f8e3119b8b2ce73ea43969eaaa3/pkg/istructs/utils.go#L35](https://github.com/voedger/voedger/blob/ae787d43bf313f8e3119b8b2ce73ea43969eaaa3/pkg/istructs/utils.go#L35)
  - Motivation:
    - Efficient CRecord caching on the DBMS side (Most CRecords reside in the same partition)
    - Simple iteration over CRecords
- **OWRecordIDSequence**: Provides sequential IDs for ORecords/WRecords (OWRecords)
  - Starts from 322680000131072
  - There are a potentially lot of such records, so it is not possible to use SELECT to read all of them

As the Voedger platform evolves, the number of sequence types is expected to expand. Future development will enable applications to define their own custom sequence types, extending the platform's flexibility to meet diverse business requirements beyond the initially implemented system sequences.

These sequences ensure consistent ordering of operations, proper transaction management, and unique identification across the platform's distributed architecture. The design prioritizes performance and scalability by implementing an efficient caching strategy and background updates that minimize memory usage and recovery time.

### Background

- [#688: record ID leads to different tables](https://github.com/voedger/voedger/issues/688)
- [VIEW RecordsRegistry](https://github.com/voedger/voedger/blob/ec85a5fed968e455eb98983cd12a0163effdc096/pkg/sys/sys.vsql#L260)
- [Singleton IDs]https://github.com/voedger/voedger/blob/ec85a5fed968e455eb98983cd12a0163effdc096/pkg/istructs/consts.go#L101

Existing design

- [const MinReservedBaseRecordID = MaxRawRecordID + 1](https://github.com/voedger/voedger/blob/84babc48d63107e29fcec28eefb0a461b5a34474/pkg/istructs/consts.go#L96)
  - 65535 + 1
  - const MaxReservedBaseRecordID = MinReservedBaseRecordID + 0xffff // 131071
  - const FirstSingletonID = MinReservedBaseRecordID // 65538
  - const MaxSingletonID = MaxReservedBaseRecordID // 66047, 512 singletons
- [ClusterAsRegisterID = 0xFFFF - 1000 + iota](https://github.com/voedger/voedger/blob/84babc48d63107e29fcec28eefb0a461b5a34474/pkg/istructs/consts.go#L69)
  - ClusterAsCRecordRegisterID
- [const FirstSingletonID](https://github.com/voedger/voedger/blob/84babc48d63107e29fcec28eefb0a461b5a34474/pkg/istructs/consts.go#L101)
- [cmdProc.appsPartitions](https://github.com/voedger/voedger/blob/b7fa6fa9e260eac4f1de80312c14ad4250f400a3/pkg/processors/command/provide.go#L32)
- [command/impl.go/getIDGenerator](https://github.com/voedger/voedger/blob/b7fa6fa9e260eac4f1de80312c14ad4250f400a3/pkg/processors/command/impl.go#L299)
- [command/impl.go: Put IDs to response](https://github.com/voedger/voedger/blob/b7fa6fa9e260eac4f1de80312c14ad4250f400a3/pkg/processors/command/impl.go#L790)
- [command/impl.go: (idGen *implIDGenerator) NextID](https://github.com/voedger/voedger/blob/b7fa6fa9e260eac4f1de80312c14ad4250f400a3/pkg/processors/command/impl.go#L816)
- [istructmem/idgenerator.go: (g *implIIDGenerator) NextID](https://github.com/voedger/voedger/blob/b7fa6fa9e260eac4f1de80312c14ad4250f400a3/pkg/istructsmem/idgenerator.go#L32)
  - [onNewID](https://github.com/voedger/voedger/blob/b7fa6fa9e260eac4f1de80312c14ad4250f400a3/pkg/istructsmem/idgenerator.go#L16)

#### Previous flow

- Recovery on the first request into the workspace
  - CP creates new `istructs.IIDGenerator` instance [here](https://github.com/voedger/voedger/blob/9d400d394607ef24012dead0d59d5b02e2766f7d/pkg/processors/command/impl.go#L136)
  - the `istructs.IIDGenerator` instance is kept for the WSID
  - `istructs.IIDGenerator` instance is tuned with the data from the each event of the PLog:
    - for each CUD:
      - CUD.ID is set as the current RecordID
        - `IIDGenerator.UpdateOnSync` is called [here](https://github.com/voedger/voedger/blob/9d400d394607ef24012dead0d59d5b02e2766f7d/pkg/processors/command/impl.go#L253)
- save the event after cmd exec:
  - `istructs.IIDGenerator` instance is provided to `IEvents.PutPlog()` [here](https://github.com/voedger/voedger/blob/9d400d394607ef24012dead0d59d5b02e2766f7d/pkg/processors/command/impl.go#L307)
  - `istructs.IIDGenerator.Next()` is called to convert rawID->realID for ODoc in arguments and each resulting CUD [here](https://github.com/voedger/voedger/blob/9d400d394607ef24012dead0d59d5b02e2766f7d/pkg/istructsmem/event-types.go#L189)

## Definitions

**APs**: Applcation Partitions

**SequencesTrustLevel**:

The `SequencesTrustLevel` setting determines how events and table records are written.

| Level | Events, write mode | Table Records, write mode |
|-------|--------------------|---------------------------|
| 0     | InsertIfNotExists  | InsertIfNotExists         |
| 1     | InsertIfNotExists  | Put                       |
| 2     | Put                | Put                       |

## Analysis

As of March 1, 2025, record ID sequences may overlap, and only 5,000,000,000 IDs are available for OWRecords, since OWRecord IDs start from 322680000131072, while CRecord IDs start from 322685000131072.

Solutions:

- **One sequence for all records**:
  - Pros:
    - 👍Clean for Voedger users
    - 👍IDs are more human-readable
    - 👍Simpler Command Processor
  - ❌Cons: CRecords are not cached efficiently
    - Solution: Let the State read copies of CRecords from sys.Collection, or possibly from an alternative optimized storage to handle large CRecord data
      - ❌Cons: Why we need CRecords then
      - 👍Pros: Separation of write and read models
- **Keep as is**:
  - Pros
    - 👍Easy to implement
  - Cons
    - ❌ No separation between write and read models
    - ❌ Only 5 billions of OWRecords (ClusterAsRegisterID < ClusterAsCRecordRegisterID)
      - Solution: Configure sequencer to use multiple ranges to avoid collisions
        - 👍Pros: Better control over sequences
  
## Solution overview

The proposed approach implements a more efficient and scalable sequence management system through the following principles:

- **Projection-Based Storage**: Each application partition will maintain sequence data in a dedicated projection ???(`SeqData`). SeqData is a map that eliminates the need to load all sequence data into memory at once
- **Offset Tracking**: `SeqData` will include a `SeqDataOffset` attribute that indicates the PLog partition offset for which the stored sequence data is valid, enabling precise recovery and synchronization
- **LRU Cache Implementation**: Sequence data will be accessed through a Most Recently Used (LRU) cache that prioritizes frequently accessed sequences while allowing less active ones to be evicted from memory
- **Background Updates**: As new events are written to the PLog, sequence data will be updated in the background, ensuring that the system maintains current sequence values without blocking operations
- **Batched Writes**: Sequence updates will be collected and written in batches to reduce I/O operations and improve throughput
- **Optimized Actualization**: The actualization process will use the stored `SeqDataOffset` to process only events since the last known valid state, dramatically reducing startup times

This approach decouples memory usage from the total number of workspaces and transforms the recovery process from a linear operation dependent on total event count to one that only needs to process recent events since the last checkpoint.

## Use cases

### VVMHost: Configure SequencesTrustLevel mode for VVM

`~tuc.VVMConfig.ConfigureSequencesTrustLevel~`uncvrd[^1]❓

VVMHost uses cmp.VVMConfig.SequencesTrustLevel.

### CP: Handling SequencesTrustLevel for Events

- `~tuc.SequencesTrustLevelForPLog~`uncvrd[^2]❓
  - When PLog is written then SequencesTrustLevel is used to determine the write mode
- `~tuc.SequencesTrustLevelForWLog~`uncvrd[^3]❓
  - When WLog is written then SequencesTrustLevel is used to determine the write mode

### CP: Handling SequencesTrustLevel for Table Records

- `~tuc.SequencesTrustLevelForRecords~`uncvrd[^4]❓
  - When a record is inserted SequencesTrustLevel is used to determine the write mode
  - When a record is updated - nothing is done in connection with SequencesTrustLevel

### CP: Command processing

- `~tuc.StartSequencesGeneration~`uncvrd[^5]❓
  - When: CP starts processing a request
  - Flow:
    - `partitionID` is calculated using request WSID and amount of partitions declared in AppDeploymentDescriptor [here](https://github.com/voedger/voedger/blob/9d400d394607ef24012dead0d59d5b02e2766f7d/pkg/vvm/impl_requesthandler.go#L61)
    - sequencer, err := IAppPartition.Sequencer(PartitionID) err
    - nextPLogOffest, ok, err := sequencer.Start(wsKind, WSID)
      - if !ok
        - Actualization is in progress
        - Flushing queue is full
        - Returns 503: "server is busy"
- `~tuc.NextSequenceNumber~`uncvrd[^6]❓
  - When: After CP starts sequences generation
  - Flow:
    - sequencer.Next(sequenceId)
- `~tuc.FlushSequenceNumbers~`uncvrd[^7]❓
  - When: After CP saves the PLog record successfully
  - Flow:
    - sequencer.Flush()
- `~tuc.ReactualizeSequences~`uncvrd[^8]❓
  - When: After CP fails to save the PLog record
  - Flow:
    - sequencer.Actualize()

### AppParts: Instantiate sequencer on Application deployment

`~tuc.InstantiateSequencer~`uncvrd[^9]❓

- When: Partition with the `partitionID` is deployed
- Flow:
  - Instantiate the implementation of the `isequencer.ISeqStorage` (appparts.internal.seqStorage, see below)
  - Instantiate `sequencer := isequencer.New(*isequencer.Params)`
  - Save `sequencer` to some `map[partitionID]isequencer.Sequencer`

---

## Technical design: Components

### IAppPartition.Sequencer

- `~cmp.IAppPartition.Sequencer~`uncvrd[^10]❓
  - Description: Returns `isequencer.ISequencer` for the given `partitionID`
  - Covers: `tuc.StartSequencesGeneration`

### VVMConfig.SequencesTrustLevel

- `~cmp.VVMConfig.SequencesTrustLevel~`uncvrd[^11]❓
  - Covers: tuc.VVMConfig.ConfigureSequencesTrustLevel

---

### pkg/isequencer

Core:

- `~cmp.ISequencer~`covered[^12]✅: Interface for working with sequences
- `~cmp.sequencer~`covered[^13]✅: Implementation of the `isequencer.ISequencer` interface
  - `~cmp.sequencer.Start~`covered[^14]✅: Starts Sequencing Transaction for the given WSID
  - `~cmp.sequencer.Next~`covered[^15]✅: Returns the next sequence number for the given SeqID
  - `~cmp.sequencer.Flush~`covered[^16]✅: Completes Sequencing Transaction
  - `~cmp.sequencer.Actualize~`covered[^17]✅: Cancels Sequencing Transaction and starts the Actualization process

Tests:

- `~test.isequencer.mockISeqStorage~`covered[^23]✅
  - Mock implementation of `isequencer.ISeqStorage` for testing purposes

Some edge case tests:

- `~test.isequencer.LongRecovery~`covered[^24]✅
  - Params.MaxNumUnflushedValues = 5 // Just a guess
  - For numOfEvents in [0, 10*Params.MaxNumUnflushedValues]
    - Create a new ISequencer instance
    - Check that Next() returns correct values after recovery
- `~test.isequencer.MultipleActualizes~`covered[^25]✅
  - Repeat { Start {Next} randomly( Flush | Actualize ) } cycle 100 times
  - Check that the system recovers well
  - Check that the sequence values are increased monotonically
- `~test.isequencer.FlushPermanentlyFails~`covered[^26]✅
  - Recovery has worked but then ISeqStorage.WriteValuesAndOffset() fails permanently
    - First Start/Flush must be ok
    - Some of the next Start must not be ok
    - Flow:
      - MaxNumUnflushedValues = 5
      - Recover
      - Mock error on WriteValuesAndOffset
      - Start/Next/Flush must be ok
      - loop Start/Next/Flush until Start() is not ok (the 6th times till unflushed values exceed the limit)  

#### interface.go

```go
/*
 * Copyright (c) 2025-present unTill Software Development Group B. V.
 * @author Maxim Geraskin
 */

package isequencer

import (
	"context"
	"time"
)

type SeqID      uint16
type WSKind     uint16
type WSID       uint64
type Number     uint64
type PLogOffset uint64

type NumberKey struct {
	WSID  WSID
	SeqID SeqID
}

type SeqValue struct {
	Key    NumberKey
	Value  Number
}

// To be injected into the ISequencer implementation.
// 
type ISeqStorage interface {

  // If number is not found, returns 0
	ReadNumbers(WSID, []SeqID) ([]Number, error)
  ReadNextPLogOffset() (PLogOffset, error)

	// IDs in batch.Values are unique
  // len(batch) may be 0
  // offset: Next offset to be used
  // batch MUST be written first, then offset
	WriteValuesAndNextPLogOffset(batch []SeqValue, nextPLogOffset PLogOffset) error

  // ActualizeSequencesFromPLog scans PLog from the given offset and send values to the batcher.
	// Values are sent per event, unordered, ISeqValue.Keys are not unique.
  // err: ctx.Err() if ctx is closed
	ActualizeSequencesFromPLog(ctx context.Context, offset PLogOffset, batcher func(batch []SeqValue, offset PLogOffset) error) (err error)
}

// ISequencer defines the interface for working with sequences.
// ISequencer methods must not be called concurrently.
// Use: { Start {Next} ( Flush | Actualize ) }
//
// Definitions
// - Sequencing Transaction: Start -> Next -> (Flush | Actualize)
// - Actualization: Making the persistent state of the sequences consistent with the PLog.
// - Flushing: Writing the accumulated sequence values to the storage.
// - LRU: Least Recently Used cache that keep the most recent next sequence values in memory.
type ISequencer interface {

  // Start starts Sequencing Transaction for the given WSID.
  // Marks Sequencing Transaction as in progress.
  // Panics if Sequencing Transaction is already started.
  // Normally returns the next PLogOffset, true
  // Returns `0, false` if:
  // - Actualization is in progress
  // - The number of unflushed values exceeds the maximum threshold
  // If ok is true, the caller must call Flush() or Actualize() to complete the Sequencing Transaction.
  Start(wsKind WSKind, wsID WSID) (plogOffset PLogOffset, ok bool)

  // Next returns the next sequence number for the given SeqID.
  // Panics if Sequencing Transaction is not in progress.
  // err: ErrUnknownSeqID if the sequence is not defined in Params.SeqTypes.
  Next(seqID SeqID) (num Number, err error)

  // Flush completes Sequencing Transaction.
  // Panics if Sequencing Transaction is not in progress.
  Flush()

  // Actualize cancels Sequencing Transaction and starts the Actualization process.
  // Panics if Actualization is already in progress.
  // Panics if Sequencing Transaction is not in progress.
  // Flow:
  // - Mark Sequencing Transaction as not in progress
  // - Cancel and wait Flushing
  // - Empty LRU
  // - Do Actualization process
  // - Write next PLogOffset
  Actualize()

}

// Params for the ISequencer implementation.
type Params struct {

	// Sequences and their initial values.
	// Only these sequences are managed by the sequencer (ref. ErrUnknownSeqID).
	SeqTypes map[WSKind]map[SeqID]Number

	SeqStorage ISeqStorage

	MaxNumUnflushedValues int           // 500
	// Size of the LRU cache, NumberKey -> Number.
	LRUCacheSize int // 100_000

  BatcherDelay time.Duration // 5 * time.Millisecond
}
```

#### Implementation requirements

```go
// filepath: pkg/isequencer: impl.go

import (
  "context"
  "time"

  "github.com/hashicorp/golang-lru/v2"
)

// Implements isequencer.ISequencer
// Keeps next (not current) values in LRU and type ISeqStorage interface
type sequencer struct {
  params *Params

  actualizerInProgress atomic.Bool

  // Set by s.Actualize(), never cleared (zeroed).
  // Used by s.cleanup().
  actualizerCtxCancel context.CancelFunc
  actualizerWG  *sync.WaitGroup


  // Cleared by s.Actualize()
  lru *lru.Cache

  // Initialized by Start()
  // Example:
  // - 4 is the offset ofthe last event in the PLog
  // - nextOffset keeps 5
  // - Start() returns 5 and increments nextOffset to 6
  nextOffset PLogOffset

  // If Sequencing Transaction is in progress then currentWSID has non-zero value.
  // Cleared by s.Actualize()
  currentWSID   WSID
  currentWSKind WSKind

  // Set by s.actualizer()
  // Closed when flusher needs to be stopped
  flusherCtxCancel context.CancelFunc
  // Used to wait for flusher goroutine to exit
  // Set to nil when flusher is not running
  // Is not accessed concurrently since 
  // - Is accessed by actualizer() and cleanup()
  // - cleanup() first shutdowns the actualizer() then flusher()
  flusherWG  *sync.WaitGroup
  // Buffered channel [1] to signal flusher to flush
  // Written (non-blocking) by Flush()
  flusherSig chan struct{}

  // To be flushed
  toBeFlushed map[NumberKey]Number
  // Will be 6 if the offset of the last processed event is 4
  toBeFlushedOffset PLogOffset
  // Protects toBeFlushed and toBeFlushedOffset
  toBeFlushedMu sync.RWMutex

  // Written by Next()
  inproc map[NumberKey]Number

}

// New creates a new instance of the Sequencer type.
// Instance has actualizer() goroutine started.
// cleanup: function to stop the actualizer.
func New(*isequencer.Params) (isequencer.ISequencer, cleanup func(), error) {
  // ...
}

// Flush implements isequencer.ISequencer.Flush.
// Flow:
//   Copy s.inproc and s.nextOffset to s.toBeFlushed and s.toBeFlushedOffset
//   Clear s.inproc
//   Increase s.nextOffset
//   Non-blocking write to s.flusherSig
func (s *sequencer) Flush() {
  // ...
}

// Next implements isequencer.ISequencer.Next.
// It ensures thread-safe access to sequence values and handles various caching layers.
// 
// Flow:
// - Validate equencing Transaction status
// - Get initialValue from s.params.SeqTypes and ensure that SeqID is known
// - Try to obtain the next value using:
//   - Try s.lru (can be evicted)
//   - Try s.inproc
//   - Try s.toBeFlushed (use s.toBeFlushedMu to synchronize)
//   - Try s.params.SeqStorage.ReadNumber()
//      - Read all known numbers for wsKind, wsID
//        - If number is 0 then initial value is used
//      - Write all numbers to s.lru
// - Write value+1 to s.lru
// - Write value+1 to s.inproc
// - Return value
func (s *sequencer) Next(seqID SeqID) (num Number, err error) {
  // ...
}

// batcher processes a batch of sequence values and writes maximum values to storage.
// 
// Flow:
// - Wait until len(s.toBeFlushed) < s.params.MaxNumUnflushedValues
//   - Lock/Unlock
//   - Wait s.params.BatcherDelay
//   - check ctx (return ctx.Err())
// - s.nextOffset = offset + 1
// - Store maxValues in s.toBeFlushed: max Number for each SeqValue.Key
// - s.toBeFlushedOffset = offset + 1
// 
func (s *sequencer) batcher(ctx ctx.Context, values []SeqValue, offset PLogOffset) (err error) {
  // ...
}

// Actualize implements isequencer.ISequencer.Actualize.
// Flow:
// - Validate Sequencing Transaction status (s.currentWSID != 0)
// - Validate Actualization status (s.actualizerInProgress is false)
// - Set s.actualizerInProgress to true
// - Set s.actualizerCtxCancel, s.actualizerWG
// - Clean s.lru, s.nextOffset, s.currentWSID, s.currentWSKind, s.toBeFlushed, s.inproc, s.toBeFlushedOffset
// - Start the actualizer() goroutine
func (s *sequencer) Actualize() {
  // ...
}

/*
actualizer is started in goroutine by Actualize().

Flow:

- if s.flusherWG is not nil
  - s.cancelFlusherCtx()
  - Wait for s.flusherWG
  - s.flusherWG = nil
- Clean s.toBeFlushed, toBeFlushedOffset
- s.flusherWG, s.flusherCtxCancel + start flusher() goroutine
- Read nextPLogOffset from s.params.SeqStorage.ReadNextPLogOffset()
- Use s.params.SeqStorage.ActualizeSequencesFromPLog() and s.batcher()
ctx handling:
 - if ctx is closed exit

Error handling:
- Handle errors with retry mechanism (500ms wait)
- Retry mechanism must check `ctx` parameter, if exists 
*/
func (s *sequencer) actualizer(ctx context.Context) {
  // ...
}

/*
flusher is started in goroutine by actualizer().

Flow:

- Wait for ctx.Done() or s.flusherSig
- if ctx.Done() exit
- Lock s.toBeFlushedMu
- Copy s.toBeFlushedOffset to flushOffset (local variable)
- Copy s.toBeFlushed to flushValues []SeqValue (local variable)
- Unlock s.toBeFlushedMu
- s.params.SeqStorage.WriteValues(flushValues, flushOffset)
- Lock s.toBeFlushedMu
- for each key in flushValues remove key from s.toBeFlushed if values are the same
- Unlock s.toBeFlushedMu

Error handling:

- Handle errors with retry mechanism (500ms wait)
- Retry mechanism must check `ctx` parameter, if exists 
*/
func (s *sequencer) flusher(ctx context.Context) {
  // ...
}

// cleanup stops the actualizer() and flusher() goroutines.
// Flow:
// - if s.actualizerInProgress
//   - s.cancelActualizerCtx()
//   - Wait for s.actualizerWG
// - if s.flusherWG is not nil
//   - s.cancelFlusherCtx()
//   - Wait for s.flusherWG
//   - s.flusherWG = nil
func (s *sequencer) cleanup() {
  // ...
}
```

---

### ISeqStorage implementation

`~cmp.ISeqStorageImplementation~`covered[^18]✅: Implementation of the `isequencer.ISeqStorage` interface

- Package: cmp.appparts.internal.seqStorage
- `~cmp.ISeqStorageImplementation.New~`covered[^19]✅
  - Per App per Partition by AppParts
  - PartitionID is not passed to the constructor
- `~cmp.ISeqStorageImplementation.i688~`covered[^20]✅
  - Handle [#688: record ID leads to different tables](https://github.com/voedger/voedger/issues/688)
  - If existing number is less than ??? 322_680_000_000_000 - do not send it to the batcher
- Uses VVMStorage Adapter

### VVMStorage Adapter

`~cmp.VVMStorageAdapter~`covered[^21]✅: Adapter that reads and writes sequence data to the VVMStorage

- Key: ((KeyPrefixSeqStorage, AppID), WSID, SeqID)
  - `~cmp.VVMStorageAdapter.KeyPrefixSeqStorage~`covered[^22]✅: Prefix for the keys in the storage

---

## Technical design: Tests

### Integration tests for SequencesTrustLevel mode

Method:

- Test for Record
  - Create a new VIT instance on an owned config with `VVMConfig.TrustedSequences = false`
  - Insert a doc to get the last recordID: simply exec `c.sys.CUD` and get the ID of the new record
  - Corrupt the storage: Insert a conflicting key that will be used on creating the next record:
    - `VIT.IAppStorageProvider.AppStorage(test1/app1).Put()`
   	- Build `pKey`, `cCols` for the record, use just inserted recordID+1
   	- Value does not matter, let it be `[]byte{1}`
  - Try to insert one more record using `c.sys.CUD`
  - Expect panic
- Test for PLog, WLog offsets - the same tests but sabotage the storage building keys for the event

Tests:

- `~it.SequencesTrustLevel0~`uncvrd[^27]❓: Intergation test for `SequencesTrustLevel = 0`
- `~it.SequencesTrustLevel1~`uncvrd[^28]❓: Intergation test for `SequencesTrustLevel = 1`
- `~it.SequencesTrustLevel2~`uncvrd[^29]❓: Intergation test for `SequencesTrustLevel = 2`

### Intergation tests for built-in sequences

- `~it.BuiltInSequences~`uncvrd[^30]❓: Test for initial values: WLogOffsetSequence, WLogOffsetSequence, CRecordIDSequence, OWRecordIDSequence

---

## Addressed issues

- [#3215: Sequences](https://github.com/voedger/voedger/issues/3215) - Initial requirements and discussion

## References

Design process:

- [Voedger Sequence Management Design (Claude 3.7 Sonnet, March 1, 2025)](https://claude.ai/chat/f1a8492a-8e8a-4229-ac79-ecc3655732d3)

History:

- [Initial design](https://github.com/voedger/voedger-internals/blob/2475814f7caa1d2d400a62a788ceda9b16d8de2a/server/design/sequences.md)

## Footnotes

[^1]: `[~server.design.sequences/tuc.VVMConfig.ConfigureSequencesTrustLevel~impl]`
[^2]: `[~server.design.sequences/tuc.SequencesTrustLevelForPLog~impl]`
[^3]: `[~server.design.sequences/tuc.SequencesTrustLevelForWLog~impl]`
[^4]: `[~server.design.sequences/tuc.SequencesTrustLevelForRecords~impl]`
[^5]: `[~server.design.sequences/tuc.StartSequencesGeneration~impl]`
[^6]: `[~server.design.sequences/tuc.NextSequenceNumber~impl]`
[^7]: `[~server.design.sequences/tuc.FlushSequenceNumbers~impl]`
[^8]: `[~server.design.sequences/tuc.ReactualizeSequences~impl]`
[^9]: `[~server.design.sequences/tuc.InstantiateSequencer~impl]`
[^10]: `[~server.design.sequences/cmp.IAppPartition.Sequencer~impl]`
[^11]: `[~server.design.sequences/cmp.VVMConfig.SequencesTrustLevel~impl]`
[^12]: `[~server.design.sequences/cmp.ISequencer~impl]` [pkg/isequencer/interface.go:47:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/isequencer/interface.go#L47)
[^13]: `[~server.design.sequences/cmp.sequencer~impl]` [pkg/isequencer/types.go:50:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/isequencer/types.go#L50)
[^14]: `[~server.design.sequences/cmp.sequencer.Start~impl]` [pkg/isequencer/impl.go:25:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/isequencer/impl.go#L25)
[^15]: `[~server.design.sequences/cmp.sequencer.Next~impl]` [pkg/isequencer/impl.go:169:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/isequencer/impl.go#L169)
[^16]: `[~server.design.sequences/cmp.sequencer.Flush~impl]` [pkg/isequencer/impl.go:263:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/isequencer/impl.go#L263)
[^17]: `[~server.design.sequences/cmp.sequencer.Actualize~impl]` [pkg/isequencer/impl.go:479:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/isequencer/impl.go#L479)
[^18]: `[~server.design.sequences/cmp.ISeqStorageImplementation~impl]` [pkg/appparts/internal/seqstorage/type.go:14:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/appparts/internal/seqstorage/type.go#L14)
[^19]: `[~server.design.sequences/cmp.ISeqStorageImplementation.New~impl]` [pkg/appparts/internal/seqstorage/provide.go:14:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/appparts/internal/seqstorage/provide.go#L14)
[^20]: `[~server.design.sequences/cmp.ISeqStorageImplementation.i688~impl]` [pkg/appparts/internal/seqstorage/impl.go:101:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/appparts/internal/seqstorage/impl.go#L101)
[^21]: `[~server.design.sequences/cmp.VVMStorageAdapter~impl]` [pkg/vvm/storage/impl_seqstorage.go:15:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/vvm/storage/impl_seqstorage.go#L15)
[^22]: `[~server.design.sequences/cmp.VVMStorageAdapter.KeyPrefixSeqStorage~impl]` [pkg/vvm/storage/consts.go:15:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/vvm/storage/consts.go#L15)
[^23]: `[~server.design.sequences/test.isequencer.mockISeqStorage~impl]` [pkg/isequencer/types.go:106:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/isequencer/types.go#L106)
[^24]: `[~server.design.sequences/test.isequencer.LongRecovery~impl]` [pkg/isequencer/isequencer_test.go:877:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/isequencer/isequencer_test.go#L877)
[^25]: `[~server.design.sequences/test.isequencer.MultipleActualizes~impl]` [pkg/isequencer/isequencer_test.go:737:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/isequencer/isequencer_test.go#L737)
[^26]: `[~server.design.sequences/test.isequencer.FlushPermanentlyFails~impl]` [pkg/isequencer/isequencer_test.go:807:impl](https://github.com/voedger/voedger/blob/851a05a0c957fcedef4ad51c3f1977abe0678a3d/pkg/isequencer/isequencer_test.go#L807)
[^27]: `[~server.design.sequences/it.SequencesTrustLevel0~impl]`
[^28]: `[~server.design.sequences/it.SequencesTrustLevel1~impl]`
[^29]: `[~server.design.sequences/it.SequencesTrustLevel2~impl]`
[^30]: `[~server.design.sequences/it.BuiltInSequences~impl]`
