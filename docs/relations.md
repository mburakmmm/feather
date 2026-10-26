# Relationship Definitions and Loading (docs/relations.md)

Feather provides declarative relationship definitions (`has_one`, `has_many`, `belongs_to`) and an explicit `RelationLoader` to prevent unexpected N+1 database queries.

## Relationship Types

1. **`has_many`:** A parent record has multiple associated child records sharing a foreign key.
   ```nox
   rel: Relation = has_many("books", "authors", "books", "author_id", "id")
   ```

2. **`belongs_to`:** A child record references a single parent record by foreign key.
   ```nox
   rel: Relation = belongs_to("author", "books", "authors", "author_id", "id")
   ```

3. **`has_one`:** A record has exactly one associated child record.
   ```nox
   rel: Relation = has_one("profile", "authors", "profiles", "author_id", "id")
   ```

## Loading Related Records with `RelationLoader`

Instead of implicit lazy properties that trigger invisible network round-trips, Feather requires explicit loading:

```nox
import feather
from feather import RelationLoader, val_int

loader: RelationLoader = RelationLoader(session)

# Load all books for author with ID 1
author_books: list[Book] = loader.load_has_many[Book](book_model, "author_id", val_int(1))

# Load the author for a given book
book_author: Author | None = loader.load_belongs_to[Author](author_model, "id", val_int(current_book.author_id))

# Load one-to-one user profile
user_profile: Profile | None = loader.load_has_one[Profile](profile_model, "user_id", val_int(current_user.id))
```

This guarantees 100% predictable query counts and transparency.
