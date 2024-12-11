---
layout: post
title:  "Utility Class for Enhancing jOOQ Queries with Spring Pagination"
date:   2024-04-19 17:00:00 +0100
categories: jekyll update
---
# Introduction
When working with jOOQ in a Java application using Spring, you might want to implement REST pagination and support 
it in the DAO layer. While jOOQ offers powerful SQL querying capabilities, it doesn’t provide built-in 
support for integrating pagination and sorting through Spring's Pageable interface.

I wrote a utility class that populates jOOQ select queries with the corresponding parameters based on the 
given Pageable object.

Below, you can find the utility class along with an example demonstrating its usage.

# Utility class
```java
import org.jooq.*;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;

import java.util.List;

/**
 * Utility class providing methods to modify jOOQ queries to include ordering, offset, and limit
 * based on the provided {@link Pageable} parameter.
 */
public final class JooqPageableUtils {

  private JooqPageableUtils() {
    throw new UnsupportedOperationException("This is a utility class and cannot be instantiated");
  }

  /**
   * Enhances a jOOQ query with pagination and sorting based on a {@link Pageable} parameter.
   *
   * <p>This method modifies the given jOOQ query to:
   *
   * <ul>
   *   <li>Apply sorting using the {@link Sort} specification in the {@link Pageable}.
   *   <li>Apply offset and limit for pagination.
   * </ul>
   *
   * @param table The jOOQ {@link Table} representing the table being queried.
   * @param select The jOOQ {@link SelectOrderByStep} query to enhance.
   * @param pageable The {@link Pageable} object containing pagination and sorting information.
   * @return The enhanced jOOQ query with sorting, offset, and limit applied.
   */
  public static SelectLimitPercentAfterOffsetStep<?> enhanceQuery(
          Table<?> table, SelectOrderByStep<?> select, Pageable pageable) {
    return select
            .orderBy(orderByClause(table, pageable))
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize());
  }

  /**
   * Creates an ORDER BY clause for the jOOQ query based on the sorting information from {@link
   * Pageable}.
   *
   * @param table The jOOQ {@link Table} to derive fields from.
   * @param pageable The {@link Pageable} object containing sorting information.
   * @return A list of jOOQ {@link Field} objects with the specified sort directions.
   */
  private static List<? extends Field<?>> orderByClause(Table<?> table, Pageable pageable) {
    return pageable.getSort().stream().map(sortOrder -> extractField(table, sortOrder)).toList();
  }

  /**
   * Extracts a jOOQ {@link Field} from the given {@link Table} based on the sorting property and
   * applies the sort direction.
   *
   * <p>This method validates the existence of the field corresponding to the sort property in the
   * {@link Table}. If the field does not exist, an {@link IllegalArgumentException} is thrown.
   *
   * @param table The jOOQ {@link Table} to derive the field from.
   * @param sortOrder The {@link Sort.Order} specifying the property and direction for sorting.
   * @return The jOOQ {@link Field} with the specified sort direction applied.
   * @throws IllegalArgumentException if the sort property does not exist in the {@link Table}.
   */
  private static Field<?> extractField(Table<?> table, Sort.Order sortOrder) {
    Field<?> field = table.field(sortOrder.getProperty());

    if (field == null) {
      throw new IllegalArgumentException("Unknown sort property: " + sortOrder.getProperty());
    }

    SortOrder order = sortOrder.isAscending() ? SortOrder.ASC : SortOrder.DESC;

    field.sort(order);
    return field;
  }
}

```
# Usage Example
In the `main` method you can see an example of `JooqPageableUtils` usage.

```java
import jooq.generated.Tables;
import jooq.generated.tables.pojos.User;
import org.jooq.DSLContext;
import org.jooq.Record;
import org.jooq.SelectJoinStep;
import org.jooq.impl.DSL;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;

import java.util.List;

import static org.springframework.data.domain.Sort.Order.asc;
import static org.springframework.data.domain.Sort.Order.desc;

public class Main {

  public static void main(String[] args) {
    DSLContext dslContext =
        DSL.using("jdbc:postgresql://localhost:5432/<db_name>", "<user>", "<password>");

    Sort sort = Sort.by(asc("created_on"), asc("status"), desc("name"));
    PageRequest pageable = PageRequest.of(2, 10, sort);

    SelectJoinStep<Record> select = dslContext.select().from(Tables.USER);

    List<User> result =
        JooqPageableUtils.enhanceQuery(Tables.USER, select, pageable).fetchInto(User.class);

    System.out.println(result);
  }
}
```

The DB that is used for this app contains only one table. Here is the DDL for the table:

```sql
CREATE TABLE "user" (
    id uuid NOT NULL PRIMARY KEY,
   "name" varchar NOT NULL,
   status varchar NOT NULL,
   created_on timestamp NOT NULL
);
```
