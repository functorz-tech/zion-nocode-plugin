# BaaS runtime — formula functions

## Formula functions (operands)
Use a function as an operand by wrapping the UPPERCASE name around its args, e.g.
{"EXTRACT_TIMESTAMPTZ": {"time": …, "unit": …}}. Args nest (a function's output can feed another when
types match). Type legend: [TYPE] required operand of that type; [TYPE?] optional (defaults 0/null);
[ANY] any type; [NUMERIC] BIGINT/INTEGER/DECIMAL/FLOAT8/BIGSERIAL; [COMPARABLE] NUMERIC/TEXT/DATE/
TIMESTAMPTZ/TIMETZ/INTERVAL; [ANY[]] array; [Enum:NAME] an explicit enum value (see Runtime enums).

Manipulation: CONCAT(items:[TEXT[]]); SUBSTRING(source_text:[TEXT],start_index:[BIGINT],end_index:
[BIGINT]); LEFT(source_text,length:[BIGINT]); RIGHT(source_text,length:[BIGINT]); LOWER(source_text);
UPPER(source_text); TRIM(text); TRIM_TRAILING_ZERO(source_text); REPEAT(text,times:[BIGINT]);
ENCODE_URL(text); DECODE_URL(text); ARRAY_CONCAT(first_array:[ANY[]],second_array:[ANY[]]);
SLICE(array:[ANY[]],start_index:[BIGINT],length:[BIGINT]); UNIQUE(array:[ANY[]]);
COALESCE(array:[ANY[]]).
Search & replace: REPLACE_OCCURRENCES(source_text,search_text,replace_text,max_replacements:
[BIGINT]); REPLACE_AT_POSITION(source_text,start_index:[BIGINT],length:[BIGINT],replace_text);
POSITION(source_text,search_text); CONTAINS(source_text,search_text).
Regex: REGEX_EXTRACT(text,regex); REGEX_REPLACE(text,regex,replacement); REGEX_EXTRACT_ALL(text,
regex); REGEX_MATCH(text,regex).
Formatting & utils: TEXT_DECIMAL_FORMAT(number:[DECIMAL],fraction_digits:[BIGINT],rounding_mode:
[Enum:ROUNDING_MODE],clear_trailing_zeros:[BOOLEAN]); NUMBER_FORMAT(number:[DECIMAL],fraction_digits,
format:[Enum:NUMBER_FORMAT]); STRING_LEN(source_text); RANDOM_TEXT(min_length,max_length,
include_numbers,include_lower_case,include_upper_case); UUID(); JOIN(array:[TEXT[]],separator:[TEXT]);
SPLIT(source_text,delimiter); ARRAY_LEN(array:[ANY[]]).
Arithmetic: ADD(value0:[NUMERIC],value1); SUBTRACT(minuend,subtrahend); MULTIPLY(value0,value1);
DIVIDE(dividend:[DECIMAL],divisor:[DECIMAL]); MODULO(dividend:[NUMERIC],divisor); ABS(number);
POW(base,exponent); LOG(base:[DECIMAL],argument:[DECIMAL]).
Rounding: ROUND_UP(number:[DECIMAL]); ROUND_DOWN(number:[DECIMAL]); DECIMAL_FORMAT(number,
fraction_digits:[BIGINT],rounding_mode:[Enum:ROUNDING_MODE]).
Generators: RANDOM_BIGINT(min_length:[BIGINT],max_length:[BIGINT]); SEQUENCE(start,end,step:[BIGINT]).
Current time: CURRENT_DATE(); CURRENT_TIMETZ(); CURRENT_TIMESTAMPTZ().
Constructors: MAKE_DATE(years,months,days:[BIGINT]); MAKE_TIMETZ(hours,minutes,seconds,
milliseconds:[BIGINT?]); MAKE_TIMESTAMPTZ(years,months,days:[BIGINT],hours,minutes,seconds,
milliseconds:[BIGINT?]); MAKE_INTERVAL(years,months,weeks,days,hours,minutes,seconds,
milliseconds:[BIGINT]); FROM_DATE_AND_TIMETZ(date:[DATE],timetz:[TIMETZ]).
Extraction & formatting: EXTRACT_DATE(time:[DATE],unit:[Enum:DATE_UNIT]); EXTRACT_TIMETZ(time:[TIMETZ],
unit:[Enum:TIME_UNIT]); EXTRACT_TIMESTAMPTZ(time:[TIMESTAMPTZ],unit:[Enum:TIMESTAMP_UNIT]);
DATE_FORMAT(time:[DATE],format:[TEXT],language:[Enum:LANGUAGE]); TIMETZ_FORMAT(time,format,language);
TIMESTAMPTZ_FORMAT(time,format,language); DURATION_FORMAT(duration:[DECIMAL],unit:[Enum:DURATION_UNIT],
format:[TEXT]); RELATIVE_DATE(time:[DATE],language,hide_suffix:[BOOLEAN]); RELATIVE_TIMESTAMPTZ(time,
language,hide_suffix).
Calculations & deltas: DELTA_DATE(date:[DATE],increase:[BOOLEAN],years,months,days:[BIGINT?]);
DELTA_TIMETZ(timetz:[TIMETZ],increase,hours,minutes,seconds,milliseconds:[BIGINT?]);
DELTA_TIMESTAMPTZ(timestamptz:[TIMESTAMPTZ],increase,years,months,days,hours,minutes,seconds,
milliseconds:[BIGINT?]); EXTRACT_DATE_DURATION(start_time,end_time:[DATE],unit:[Enum:DATE_UNIT]);
EXTRACT_TIMETZ_DURATION(start_time,end_time:[TIMETZ],unit:[Enum:TIME_UNIT]);
EXTRACT_TIMESTAMPTZ_DURATION(start_time,end_time:[TIMESTAMPTZ],unit:[Enum:TIMESTAMP_UNIT]).
Conversions: FROM_TIMESTAMPTZ_TO_DATE(timestamptz); FROM_TIMESTAMPTZ_TO_TIMETZ(timestamptz).
Geography: FROM_COORDINATES(latitude:[DECIMAL],longitude:[DECIMAL]); GEO_DISTANCE(point0,point1:
[GEO_POINT],unit:[Enum:GEO_DISTANCE_UNIT]); GEO_LONGITUDE(geo:[GEO_POINT]); GEO_LATITUDE(geo).
Aggregates: SUM(array:[NUMERIC[]]); AVG(array:[NUMERIC[]]); MAX(value0,value1:[COMPARABLE]);
MIN(value0,value1); GREATEST(array:[COMPARABLE[]]); LEAST(array:[COMPARABLE[]]).
Element access: ITEM(array:[ANY[]],index:[BIGINT]); FIRST_ITEM(array); LAST_ITEM(array);
RANDOM_ITEM(array); ARRAY_POSITION(array:[ANY[]],item:[ANY]).
JSONB: JSON_EXTRACT_BY_DOT_NOTATION_JSONPATH(json:[JSONB],path:[TEXT]).
Casting: CAST_FROM_TEXT(value:[TEXT]); CAST_COLUMN_TO_TEXT(value:[ANY]); CAST_ARRAY_TO_TEXT(value:
[ANY[]]); CAST_TO_BIGINT(value:[ANY]); CAST_TO_DECIMAL(value:[ANY]).
Vector search: EMBEDDING_VECTOR_DISTANCE(embedded_text_column:[Enum:EMBEDDING_COLUMN],text:[TEXT],
distance_function:[Enum:VECTOR_DISTANCE]).
System: NULL_VALUE().

## Runtime enums (exact string values for [Enum:NAME] operands)
- DATE_UNIT: YEAR, MONTH, DAY, WEEK
- DURATION_UNIT: DAY, HOUR, MINUTE, SECOND, MILLISECOND
- EMBEDDING_COLUMN: (generated per column with the TEXT_COLUMN_VECTOR_SORT extension)
- GEO_DISTANCE_UNIT: METER, KILOMETER, MILE
- LANGUAGE: EN, ZH
- NUMBER_FORMAT: THOUSANDS_SEPARATOR, PERCENT
- ROUNDING_MODE: HALF_EVEN, HALF_UP, HALF_DOWN, UP, DOWN, CEILING, FLOOR
- TIMESTAMP_UNIT: YEAR, MONTH, DAY, HOUR, MINUTE, SECOND, MILLISECOND, WEEK
- TIME_UNIT: HOUR, MINUTE, SECOND, MILLISECOND
- VECTOR_DISTANCE: EUCLIDEAN, COSINE

## Testing against the deployed backend (CLI)

This is a **runtime** spoke — it describes calling a DEPLOYED Zion app's SINGLE auto-generated
GraphQL API, which exposes ALL backend interactions (database, action flows, third-party APIs, AI
agents), not editing the editor schema. Endpoints (`{projectExId}` = the project's external id):
- HTTP (queries + mutations): https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2
- WebSocket (subscriptions):  wss://zion-app.functorz.com/zero/{projectExId}/api/graphql-subscription

Exercise runtime queries/mutations straight from this CLI — already authenticated with the admin token:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" support graphql --args '{"query":"query { <root_op> { ... } }","variables":{}}'
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" support query   --args '{"tableName":"post","where":{"id":{"_eq":1}},"limit":20,"fields":["id","title"]}'
```
`support graphql` sends **raw** GraphQL (use the operator-first `where` grammar in `baas-database.md`); `support query/insert/update/delete` are typed helpers that take the **simplified** `where` (see `schema-table.md`). Subscriptions (async action-flow results, AI streaming) run from your generated frontend over the WebSocket endpoint — this CLI does not open runtime subscriptions.

These are **GraphQL operand** functions used inside `baas-database.md` filters/sorts — distinct from the **UI formula operators** in `data-binding.md`; do not conflate the two.
