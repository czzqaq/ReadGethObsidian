# OnEvent
```go
	switch x := ev.(type) {
	case engine.BuildStartedEvent:
		d.onBuildStarted(x)
	case engine.InvalidPayloadAttributesEvent:
		d.onInvalidPayloadAttributes(x)
	case engine.BuildSealedEvent:
		d.onBuildSealed(x)
	case engine.PayloadSealInvalidEvent:
		d.onPayloadSealInvalid(x)
	case engine.PayloadSealExpiredErrorEvent:
		d.onPayloadSealExpiredError(x)
	case engine.PayloadInvalidEvent:
		d.onPayloadInvalid(x)
	case engine.PayloadSuccessEvent:
		d.onPayloadSuccess(x)
	case SequencerActionEvent:
		d.onSequencerAction(x)
	case rollup.EngineTemporaryErrorEvent:
		d.onEngineTemporaryError(x)
	case rollup.ResetEvent:
		d.onReset(x)
	case engine.EngineResetConfirmedEvent:
		d.onEngineResetConfirmedEvent(x)
	case engine.ForkchoiceUpdateEvent:
		d.onForkchoiceUpdate(x)
```

这些事件的触发者不尽相同，被 engine 管理。下面就说明这些事件的含义，以及事件触发后，sequencer 做了什么
## 各个事件

### BuildStartedEvent


