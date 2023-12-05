# discussion items for the next cue collaboration meeting

## `cue fmt`  issues

### add `TestParseFmtIssues` tests?

- added tests `func TestParseFmtIssues` that pass with the current buggy results, see [parser_test.go](https://github.com/rudifa/cue/commit/2ff2db3e90c388f2e1e354669cf777ca0d275d35) ; they will fail when the bugs are fixed
- could add the whitespace and indent case tests

### drill in  to find the cause of the bugs?

- could drill into the FileParser and see if I can figure out why is it mishandling comments
- would need to understand ast.File: how are comments handled, encoded, etc.
- would need to understand the parser: how are comments handled, encoded, etc.
- would need to understand `func DebugStr`: does it show all pertinent detals?

## issue #2704 (and #2354 and #2209)

[v0.6.0 cue vet now failing validation when v0.4.3 was passing #2704](https://github.com/cue-lang/cue/issues/2704)

_This issue reminds me slightly of #2354, which is also an ordering issue with disjunctions. It requires a default value and doesn't use regular expressions, though._

[evaluator: order of disjunction with default case seems to affect if comprehensions #2354](https://github.com/cue-lang/cue/issues/2704)

_This bug reminds me of #2209 slightly; in that case, removing a disjunction in a definition changed the output in an unexpected and buggy way. However, that case worked on v0.4.3, so I don't think it's a duplicate._

[evaluator: 0.5 regression with disjunctions and comprehensions #2209](https://github.com/cue-lang/cue/issues/2209))

### bisected #2704 to commit e7cfb50 by @mpvl

```
cue % git bisect good                                  [:cabc1db|BISECT 2; 1 steps L|…1]
e7cfb50772304833aaf8ac4471aa99638fe1a5a6 is the first bad commit
commit e7cfb50772304833aaf8ac4471aa99638fe1a5a6
Author: Marcel van Lohuizen <mpvl@gmail.com>
Date:   Tue Jul 25 12:15:11 2023 +0200
    internal/core/adt: do not delay processing of fields
...
    Fixes #2351
    Fixes #2355
...
 99 files changed, 686 insertions(+), 742 deletions(-)
```

Of the 99 files changed, only 2 files are go files, others are .txtar files:

```
    internal/core/adt/composite.go      // 2 changes
    internal/core/adt/eval.go           // 14 changes
```

- simpler reproducer
- shows that reversing the order of the disjunct in the schema makes the problem go away

### curioser and curioser

- 3900 calls to BinOp in the failing case, 3886 in the passing case

```
var binOpCount int

// BinOp handles all operations except AndOp and OrOp. This includes processing
// unary comparators such as '<4' and '=~"foo"'.
//
// BinOp returns nil if not both left and right are concrete.
func BinOp(c *OpContext, op Op, left, right Value) Value {
 leftKind := left.Kind()
 rightKind := right.Kind()

 binOpCount++
 fmt.Printf("BinOp: count= %d op= %s left= %s right= %s\n", binOpCount, op, leftKind, rightKind)
```

#### the failing case

```

BinOp: count= 3896 op= =~ left= string right= string
BinOp: MatchOp: c.regexp(right)= &regexp.Regexp{expr:"^[+-]?[0-9]+$", prog:(*syntax.Prog)(0xc0002e2330), onepass:(*regexp.onePassProg)(0xc0002e2390), numSubexp:0, maxBitStateLen:0, subexpNames:[]string{""}, prefix:"", prefixBytes:[]uint8(nil), prefixRune:0, prefixEnd:0x1, mpool:0, matchcap:2, prefixComplete:false, cond:0x4, minInputLen:1, longest:false}
BinOp: MatchOp: c.stringValue(left, op)= "5"
BinOp: count= 3897 op= =~ left= string right= string
BinOp: MatchOp: c.regexp(right)= &regexp.Regexp{expr:"^[+-]?[0-9]+$", prog:(*syntax.Prog)(0xc0002e2330), onepass:(*regexp.onePassProg)(0xc0002e2390), numSubexp:0, maxBitStateLen:0, subexpNames:[]string{""}, prefix:"", prefixBytes:[]uint8(nil), prefixRune:0, prefixEnd:0x1, mpool:0, matchcap:2, prefixComplete:false, cond:0x4, minInputLen:1, longest:false}
BinOp: MatchOp: c.stringValue(left, op)= "5"
BinOp: count= 3898 op= == left= string right= string
BinOp: count= 3899 op= =~ left= string right= string
BinOp: MatchOp: c.regexp(right)= &regexp.Regexp{expr:"^[+-]?[0-9]+$", prog:(*syntax.Prog)(0xc0002e2330), onepass:(*regexp.onePassProg)(0xc0002e2390), numSubexp:0, maxBitStateLen:0, subexpNames:[]string{""}, prefix:"", prefixBytes:[]uint8(nil), prefixRune:0, prefixEnd:0x1, mpool:0, matchcap:2, prefixComplete:false, cond:0x4, minInputLen:1, longest:false}
BinOp: MatchOp: c.stringValue(left, op)= "value"
BinOp: count= 3900 op= =~ left= string right= string
BinOp: MatchOp: c.regexp(right)= &regexp.Regexp{expr:"^[+-]?[0-9]+$", prog:(*syntax.Prog)(0xc0002e2330), onepass:(*regexp.onePassProg)(0xc0002e2390), numSubexp:0, maxBitStateLen:0, subexpNames:[]string{""}, prefix:"", prefixBytes:[]uint8(nil), prefixRune:0, prefixEnd:0x1, mpool:0, matchcap:2, prefixComplete:false, cond:0x4, minInputLen:1, longest:false}
BinOp: MatchOp: c.stringValue(left, op)= "value"
1.settingC: invalid value "value" (out of bound =~"^[+-]?[0-9]+$"):
    ./testdata/2704-3.cue:3:15
    ./testdata/2704-3.json:6:29

```

#### the passing case

```

BinOp: count= 3895 op= =~ left= string right= string
BinOp: MatchOp: c.regexp(right)= &regexp.Regexp{expr:"^[+-]?[0-9]+$", prog:(*syntax.Prog)(0xc000329f80), onepass:(*regexp.onePassProg)(0xc000362000), numSubexp:0, maxBitStateLen:0, subexpNames:[]string{""}, prefix:"", prefixBytes:[]uint8(nil), prefixRune:0, prefixEnd:0x1, mpool:0, matchcap:2, prefixComplete:false, cond:0x4, minInputLen:1, longest:false}
BinOp: MatchOp: c.stringValue(left, op)= "5"
BinOp: count= 3896 op= =~ left= string right= string
BinOp: MatchOp: c.regexp(right)= &regexp.Regexp{expr:"^[+-]?[0-9]+$", prog:(*syntax.Prog)(0xc000329f80), onepass:(*regexp.onePassProg)(0xc000362000), numSubexp:0, maxBitStateLen:0, subexpNames:[]string{""}, prefix:"", prefixBytes:[]uint8(nil), prefixRune:0, prefixEnd:0x1, mpool:0, matchcap:2, prefixComplete:false, cond:0x4, minInputLen:1, longest:false}
BinOp: MatchOp: c.stringValue(left, op)= "value"
```

## [Boris Beizer quotes](https://www.azquotes.com/author/44000-Boris_Beizer)

> Bugs lurk in corners and congregate at boundaries.
