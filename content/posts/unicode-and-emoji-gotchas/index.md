+++
title = 'Fully supporting Unicode and emojis in your app'
description = "Common issues with Unicode and emoji support"
image = "unicode-emoji.png"
date = 2024-02-29
tags = ["Unicode", "Emoji", "MySQL"]
categories = []
draft = false
+++

## Introduction

Any app aiming to reach an international audience must support Unicode. Emojis, which are based on Unicode, are everywhere. They are used in text messages, social media, and programming languages. Supporting Unicode and emojis in your app can be tricky. This article will cover common Unicode and emoji support issues and how to fix them.

## What is Unicode?

Unicode is a standard for encoding, representing, and handling text. It is a character set that assigns a unique number to every character. The most common encoding for Unicode is UTF-8, which stands for Unicode Transformation Format 8-bit. UTF-8 is a variable-width encoding that can represent every character in the Unicode character set.

UTF-8 format can take one to four bytes to represent a code point. Multiple code points can be combined to form a single character. For example, the emoji "👍" is represented by the code point `U+1F44D`. In UTF-8, it is represented by the bytes `F0 9F 91 8D`. The same emoji with skin tone "👍🏽" is represented by the code point `U+1F44D U+1F3FD`. In UTF-8, that emoji is represented by the bytes `F0 9F 91 8D F0 9F 8F BD`. Generally, emojis take up at least four bytes in UTF-8.

## Unicode equivalence

Our first gotcha is unicode equivalence.

Unicode equivalence is the concept that two different sequences of code points can represent the same character. For example, the character `é` can be represented by the code point `U+00E9` or by the sequence of code points `U+0065 U+0301`. The first representation is the composed form, and the second is the decomposed form. Unicode equivalence is essential when comparing strings or searching for a string character.

Databases typically do not support Unicode equivalence out of the box. For example, given this table using MySQL 5.7:

```sql
CREATE TABLE test (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    PRIMARY KEY (id))
    CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

INSERT INTO test (name) VALUES ('가'), (CONCAT('ᄀ', 'ᅡ'));

SELECT * from test WHERE name = '가';
```

The query will return a single row, even though the Korean character `가` and character sequence `ᄀ` + `ᅡ` are equivalent. The incorrect result is because the `utf8mb4_unicode_ci` collation does not support Unicode equivalence. One way to fix this is to use the `utf8mb4_0900_ai_ci` collation, which supports Unicode equivalence. However, this requires updating the database to MySQL 8.0 or later, which may not be possible in some cases.

## Emoji equivalence

Our second gotcha is emoji equivalence.

Some databases may not support emoji equivalence out of the box. For example, given this table using MySQL 5.7:

```sql
CREATE TABLE test (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    PRIMARY KEY (id))
    CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

INSERT INTO test (name) VALUES ('🔥'), ('🔥🔥'), ('👍'), ('👍🏽');

SELECT * from test WHERE name = '🔥';
```

The query will return:
```
1,🔥
3,👍
```

And the following query:

```sql
SELECT * from test WHERE name LIKE '%🔥%';
```

Will return:

```
1,🔥
2,🔥🔥
```

The `utf8mb4_unicode_ci` collation does not support emoji equivalence, and the behavior of `=` differs from `LIKE.`

One way to fix the problem of emoji equivalence is to use a different collation during the `=` comparison. For example:

```sql
SELECT * from test WHERE name COLLATE utf8mb4_bin = '🔥';
```

Will return the single correct result:
```
1,🔥
```

However, this solution is not ideal because it requires the developer to remember to use the `utf8mb4_bin` collation for emoji equivalence. There is also a slight performance impact when using a different collation.

## Case-insensitive sorting

Our third gotcha is sorting.

Typically, app users want to see case-insensitive sorting of strings. For example, the strings "apple", "Banana", and "cherry" should be sorted as "apple", "Banana", and "cherry". The `utf8mb4_unicode_ci` collation used above supports case-insensitive sorting. However, switching to another collation, such as `utf8mb4_bin`, to support emoji equivalence will break case-insensitive sorting. Hence, whatever solution you develop for full Unicode support should also support case-insensitive sorting.

## Solving our gotchas with normalization

A partial solution to the above gotchas is to use normalization. Normalization is the process of transforming text into a standard form. Unicode defines four normalization forms: NFC, NFD, NFKC, and NFKD. The most common normalization form is NFC, which is the composed form. NFC is the standard form for most text processing.

For example, in the following Go code:

```go
package main

import (
    "fmt"
    "golang.org/x/text/unicode/norm"
    "strconv"
)

func main() {
    str1, _ := strconv.Unquote(`"\uAC00"`)       // 가
    str2, _ := strconv.Unquote(`"\u1100\u1161"`) // ᄀ + ᅡ
    fmt.Println(str1)
    fmt.Println(str2)
    if str1 == str2 {
       fmt.Println("raw equal")
    } else {
       fmt.Println("raw not equal")
    }
    strNorm1 := norm.NFC.String(str1)
    strNorm2 := norm.NFC.String(str2)
    if strNorm1 == strNorm2 {
       fmt.Println("normalized equal")
    } else {
       fmt.Println("normalized not equal")
    }

}
```

The two strings are not equal in their raw form but equal after normalization. Normalizing before inserting, updating, and searching in the database can solve the Unicode equivalence issue while allowing the user to keep the case-insensitive sorting.

To solve emoji equivalence, we can use the `utf8mb4_bin` collation for the `=` comparison. However, if our column is indexed, we may need to use the `utf8mb4_bin` collation for the index. We cannot have a different collation for the column and the index, but we could use a second generated column with the `utf8mb4_bin` collation and index that column.

## Conclusion

Unicode and emoji support is essential for any app aiming to reach an international audience. Unicode equivalence, emoji equivalence, and case-insensitive sorting are common issues with Unicode and emoji support. Normalization can solve the Unicode equivalence issue while allowing the user to keep the case-insensitive sorting. Using the `utf8mb4_bin` collation for the `=` comparison can solve the emoji equivalence issue.

## Fully supporting Unicode and emojis in your app video

{{< youtube u9jFFHifa0Q >}}

## Other articles related to MySQL

- [Optimize MySQL query performance: INSERT with subqueries](../mysql-query-performance-insert-subqueries/)
- [MySQL deadlock on UPDATE/INSERT upsert pattern](../mysql-upsert-deadlock/)
- [SQL prepared statements are broken when scaling applications](../sql-prepared-statements-are-broken-when-scaling-applications/)

*Note:* If you want to comment on this article, please do so on the YouTube video.
