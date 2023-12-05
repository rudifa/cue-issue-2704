# [issue #2704](https://github.com/cue-lang/cue/issues/2704)

[v0.6.0 cue vet now failing validation when v0.4.3 was passing #2704](https://github.com/cue-lang/cue/issues/2704)

This issue reminds me slightly of #2354, which is also an ordering issue with disjunctions. It requires a default value and doesn't use regular expressions, though. @mvdan

[evaluator: order of disjunction with default case seems to affect if comprehensions #2354](https://github.com/cue-lang/cue/issues/2354)

This bug reminds me of #2209 slightly; in that case, removing a disjunction in a definition changed the output in an unexpected and buggy way. However, that case worked on v0.4.3, so I don't think it's a duplicate. @mvdan

[evaluator: 0.5 regression with disjunctions and comprehensions #2209](https://github.com/cue-lang/cue/issues/2209)
