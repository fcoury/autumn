# Autumn, a Python ORM

Autumn exists as a super-lightweight Object-relational mapper (ORM) for Python. 
It’s an alternative to [SQLObject](http://www.sqlobject.org/), 
[SQLAlchemy](http://www.sqlalchemy.org/), [Storm](https://storm.canonical.com/),
etc. Perhaps the biggest difference is the automatic population of fields as 
attributes (see the example below).

It is released under the MIT License (see LICENSE file for details).

This project is currently considered beta software.

## Special additions of this fork

Changes I made to the ORM inner workings:

### 1. Table names are now pluralized by default

I have implemented a default (and so far very weak) pluralization for table names, and replicated the Rails parsing method. 

So now, this code:

    class OrderItem(Model):
		class Meta:
			pass
			
would get tied to the `order_items` table, and not `orderitem` as it would on the master branch of autumn.

### 2. Added find method that returns always a model instance, not a list

On master branch, the only way to ask for models is through the get method. Right now `Model.get(id)` returns an instance of the model and `Model.get(attribute=attr_value)` returns a list of values.

Again, in order to match something we have in Rails (sort-of anyways), I have created a similar method, `find()`, that does not return a list, so

	author = Author.find(first_name='Alex')

Will always return the *first* author whose name is "Alex". That is useful for not needing to do [0] expansion. This will be refactored on the future.

### 3. Added support for WHERE..IN clause when values are arrays
	
This code:

	a = Author.find(first_name=['Alex', 'Mike', 'Charles'])
	b = Author.get(first_name=['Alex', 'Mike', 'Charles'])  

Now renders the following SQL:

	SELECT * FROM authors WHERE first_name IN ('Alex', 'Mike', 'Charles')

## MySQL Example

Using these tables:

    DROP TABLE IF EXISTS author;
    CREATE TABLE author (
        id INT(11) NOT NULL auto_increment,
        first_name VARCHAR(40) NOT NULL,
        last_name VARCHAR(40) NOT NULL,
        bio TEXT,
        PRIMARY KEY (id)
    );
    DROP TABLE IF EXISTS books;
    CREATE TABLE books (
        id INT(11) NOT NULL auto_increment,
        title VARCHAR(255),
        author_id INT(11),
        FOREIGN KEY (author_id) REFERENCES author(id),
        PRIMARY KEY (id)
    );

We setup our objects like so:

    from autumn.db.connection import db
    from autumn.model import Model
    from autumn.db.relations import ForeignKey, OneToMany
    import datetime

    db.connect('mysql', user='root', db='mydatabase')

    class Author(Model):
        books = OneToMany('Book')

        class Meta:
            defaults = {'bio': 'No bio available'}
            validations = {'first_name': lambda self, v: len(v) > 1}

    class Book(Model):
        author = ForeignKey(Author)

        class Meta:
            table = 'books'

Now we can create, retrieve, update and delete entries in our database.
Creation

    james = Author(first_name='James', last_name='Joyce')
    james.save()

    u = Book(title='Ulysses', author_id=james.id)
    u.save()

### Retrieval

    a = Author.get(1)
    a.first_name # James
    a.books      # Returns list of author's books

    # Returns a list, using LIMIT based on slice
    a = Author.get()[:10]   # LIMIT 0, 10
    a = Author.get()[20:30] # LIMIT 20, 10

### Updating

    a = Author.get(1)
    a.bio = 'What a crazy guy! Hard to read but... wow!'
    a.save()

### Deleting

    a.delete()

