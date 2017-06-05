---
layout: post
title: "Regular Expression"
categories: blog
excerpt: "Regular Expression Summary"
tags: [Regex]
share: true
image:
  feature:
date: 2017-02-19T18:13:05-08:00
modified: 
---

#### POSIX BRE and ERE metacharacters

| Character | BRE/ERE | Meaning in a pattern |
|:----------|:-------:|---------------------:|
|     \     |  Both   | Turn off the special meaning of the following character. Occasionally, enable aspecial meaning for the following character in BRE, such as for \(…\) and \{…\}.   |
|     .     |  Both   | Match any single character except NUL. Individual programs may also disallow matching newline.   |
|     *     |  Both   | Match any number (or none) of the single character that immediately precedes it. For EREs, the preceding character can instead be a regular expression. For example, since . (dot) means any character, .* means “match any number of any character.” For BREs, * is not special if it’s the first character of a regular expression.   |
|     ^     |  Both   | Match the following regular expression at the beginning of the line or string. BRE: special only at the beginning of a regular expression. ERE: special everywhere.   |
|     $     |  Both   | Match the preceding regular expression at the end of the line or string. BRE: special only at the end of a regular expression. ERE: special everywhere.   |
|   [...]   |  Both   | Termed a bracket expression, this matches any one of the enclosed characters. A hyphen (-) indicates a range of consecutive characters. (Caution: ranges are locale-sensitive, and thus not portable.) A circumflex (^) as the first character in the brackets reverses the sense: it matches any one character not in the list. A hyphen or close bracket (]) as the first character is treated as a member of the list. All other metacharacters are treated as members of the list (i.e., literally). Bracket expressions may contain collating symbols, equivalence classes, and character classes (described shortly). |
| \\{n,m\\} |   BRE   | Termed an interval expression, this matches a range of occurrences of the single character that immediately precedes it. \{n\} matches exactly n occurrences, \{n,\} matches at least n occurrences, and \{n,m\} matches any number of occurrences between n andm. n andm must be between 0 and RE_DUP_MAX (minimum value: 255), inclusive. |
| \\(...\\) |   BRE   | Save the pattern enclosed between \\( and \\) in a special holding space. Up to nine subpatterns can be saved on a single pattern. The text matched by the subpatterns can be reused later in the same pattern, by the escape sequences \1 to \9. For example, \\(ab\\).*\1 matches two occurrences of ab, with any number of characters in between. |
|    \\n    |   BRE   | Replay the nth subpattern enclosed in \\( and \\) into the pattern at this point. n is a number from 1 to 9, with 1 starting on the left.
|   {n,m}   |   ERE   | Just like the BRE \\{n,m\\} earlier, but without the backslashes in front of the braces. |
|     +     |   ERE   | Match one or more instances of the preceding regular expression. |
|     ?     |   ERE   | Match zero or one instances of the preceding regular expression. |
|    \|     |   ERE   | Match the regular expression specified before or after. |
|   (...)   |   ERE   | Apply a match to the enclosed group of regular expressions. |
{: .table}

#### Additional POSIX bracket expressions

*Character classes*  
A POSIX character class consists of keywords bracketed by [: and :]. The keywords
describe different classes of characters such as alphabetic characters,control
characters, and so on. See Table 3-3.

*Collating symbols*  
A collating symbol is a multicharacter sequence that should be treated as a unit.
It consists of the characters bracketed by [. and .]. Collating symbols are specific
to the locale in which they are used.

*Equivalence classes*  
An equivalence class lists a set of characters that should be considered equivalent,
such as e and è. It consists of a named element from the locale,bracketed
by [= and =].

>For example, [[:alpha:]!] matches any single alphabetic character or
>the exclamation mark,and [[.ch.]] matches the collating element ch,but does not
>match just the letter c or the letter h. In a French locale, [[=e=]] might match any of
>e, è, ë, ê,or é.

*POSIX character classes*

|   Class   |   Matching characters   |   Class    |  Matching characters   |
|:----------|:-----------------------:|-----------:|-----------------------:|
| [:alnum:] | Alphanumeric characters | [:lower:]  | Lowercase characters   |
| [:alpha:] | Alphabetic characters   | [:print:]  | Printable characters   |
| [:blank:] | Space and tab characters| [:punct:]  | Punctuation characters |
| [:cntrl:] | Control characters      | [:space:]  | Whitespace characters  |
| [:digit:] | Numeric characters      | [:upper:]  | Uppercase characters   |
| [:graph:] | Nonspace characters     | [:xdigit:] | Hexadecimal digits     |
{: .table}

#### operator precedence

*BRE operator precedence from highest to lowest Operator Meaning*

|    Operator    |         Meaning         |
|:---------------|:-----------------------:|
| [..] [==] [::] | Bracket symbols for character collation |
| \metacharacter | Escaped metacharacters |
|      []        | Bracket expressions |
|  \\(\\) \digit | Subexpressions and backreferences |
|   * \\{\\}     | Repetition of the preceding single-character regular expression |
|   no symbol    | Concatenation |
|      ^ $       | Anchors |
{: .table}

*ERE operator precedence from highest to lowest*

|    Operator    |         Meaning         |
|:---------------|:-----------------------:|
| [..] [==] [::] | Bracket symbols for character collation |
| \metacharacter | Escaped metacharacters |
|      []        | Bracket expressions |
|      ()        | Grouping |
|   * + ? {}     | Repetition of the preceding regular expression |
|   no symbol    | Concatenation |
|      ^ $       | Anchors |
|      \|        | Alternation |
{: .table}

#### Regular Expression Extensions

*Additional GNU regular expression operators*

|    Operator    |         Meaning         |
|:---------------|:-----------------------:|
|       \w       | Matches any word-constituent character. Equivalent to [[:alnum:]_]. |
|       \W       | Matches any nonword-constituent character. Equivalent to [^[:alnum:]_]. |
|    \\< \\>     | Matches the beginning and end of a word, as described previously. |
|       \b       | Matches the null string found at either the beginning or the end of a word. This is a generalization of the \< and \> operators. Note: Because awk uses \b to represent the backspace character, GNU awk (gawk) uses \y. |
|       \B       | Matches the null string between two word-constituent characters. |
|    \\' \\`     | Matches the beginning and end of an emacs buffer, respectively. GNU programs (besides emacs) generally treat these as being equivalent to ^ and $. |
{: .table}
