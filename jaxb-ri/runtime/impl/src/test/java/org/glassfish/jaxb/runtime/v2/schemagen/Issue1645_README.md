# Issue #1645 - JAXB Indentation Fix Verification

## Overview
This document verifies that issue [#1645](https://github.com/eclipse-ee4j/jaxb-ri/issues/1645) has been fixed in the current codebase.

## Problem Description
The issue reported that `JAXB_FORMATTED_OUTPUT` was creating strange output when there were more than eight levels of nesting. The indentation would not work properly beyond 8 levels.

## Root Cause
The bug was in `IndentingUTF8XmlOutput.java` in the `printIndent()` method. The code was incorrectly calculating the number of complete repetitions of the 8-indentation buffer:

### Before (Buggy Code):
```java
int i = depth%8;
write( indent8.buf, 0, i*unitLen );
i>>=3;  // This was dividing the modulo result (0-7) by 8, always resulting in 0!
for( ; i>0; i-- )
    indent8.write(this);
```

The problem: `i>>=3` was shifting the modulo result (which is always 0-7), not the original `depth` value. This meant the loop would never execute for depths > 8.

### After (Fixed Code):
```java
int i = depth & 0x7;    // modulo 8
write( indent8.buf, 0, i*unitLen );
i = depth>>3;           // This correctly divides depth by 8!
for( ; i>0; --i )
    indent8.write(this);
```

The fix: Calculate the number of full buffer repetitions directly from `depth`, not from the modulo result.

## Fix Location
**File:** `/jaxb-ri/runtime/impl/src/main/java/org/glassfish/jaxb/runtime/v2/runtime/output/IndentingUTF8XmlOutput.java`

**Method:** `printIndent()` (lines 110-120)

**Line:** 116

## Test Coverage
A comprehensive test suite has been added in `IndentationTest.java` that verifies:

1. **8-level indentation (boundary condition)**: Tests exactly 8 levels of nesting
2. **11-level indentation**: Tests deep nesting beyond the 8-level threshold
3. **16-level indentation**: Tests double the buffer size to ensure scalability

All tests pass successfully with the current implementation.

## Test Results
The test output shows proper indentation at all levels:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<level0>
    <level1>
        <level2>
            <level3>
                <level4>
                    <level5>
                        <level6>
                            <level7>
                                <level8>
                                    <level9>
                                        <level10>
                                            <content>Deep Content</content>
                                        </level10>
                                    </level9>
                                </level8>
                            </level7>
                        </level6>
                    </level5>
                </level4>
            </level3>
        </level2>
    </level1>
</level0>
```

Notice that level 9, 10, and the content within level 10 are all properly indented with the correct number of spaces (36, 40, and 44 spaces respectively).

## Running the Tests
To run the test suite:

```bash
cd jaxb-ri/runtime/impl
mvn test -Dtest=IndentationTest
```

All tests should pass with output demonstrating proper indentation at all levels.

## Conclusion
Issue #1645 has been successfully fixed in the current master branch. The fix correctly handles indentation for any depth of nesting, not just up to 8 levels. The test suite provides comprehensive coverage to prevent regression of this bug.
