# 1. Kapcsolódás MongoDB adatbázishoz

- MongoDB Shell
  ```
  mongosh // connects to mongodb://127.0.0.1:27017 by default<br>
  mongosh "mongodb://<user>:<password>@192.168.1.1:27017"<br>
  mongosh "mongodb+srv://<user>:<password>@sandbox.3fiqf.mongodb.net/"
  ```
- Studio 3T Intelli Shell
  ```
  Connection string: mongodb+srv://<username>:<password>@sandbox.3fiqf.mongodb.net/
  ```

# 2. Listázás, kijelölés

- Adatbázisok listázása
  ```
  show dbs
  ```
- Aktuális adatbázis kijelölése (nem kell, hogy az adatbázis létezzen)
  ```
  use <adatbázis neve
  ```
- Aktuális adatbázis neve
  ```
  db
  ```
- Kollekciók (táblák) listája az aktuális adatbázisban:
  ```
  show collections
  ```
- Kollekcióban lévő dokumentum(ok) listázása
  ```
  db.kollekció.find()
  ```

# 3. CRUD Műveletek

## 3.1 Dokumentum létehozás (Create, nem kell, hogy a tábla ("teszt") létezzen)

> A MongoDB alapértelmezett elsődleges kulcsa "\_id" nevet kap<br>
> Ha nem adjuk meg egyedi értékét, akkor a MongoDB hozza létre<br>
> Pl.: \_id: ObjectId("65e46484a61c9ec18b2c72ea"),

- Egy dokumentum (rekord) beszúrása
  ```
  db.teszt.insertOne({nev: "Szabó János", kor: NumberInt(34)})
  ```
- Több dokumentum beszúrása
  ```
  db.teszt.insertMany([{nev: "Kiss Péter", kor: NumberInt(41)}, {nev: "Török Béla", kor: NumberInt(55)}])
  ```
- Függvények használata
  > NumberInt(34) -> Függvény nélkül valós típusú (34.0) lenne az érték
  ```
  db.teszt.insertOne({dátum: ISODate()}), azonos a következő paranccsal:
  db.teszt.insertOne({dátum: new Date()})
  Aktuális GMT időt szúr be, Magyarország GMT+1 (téli), vagy GMT+2 (nyári) időzónát használ
  db.teszt.insertOne({dátum: new Date("2024-03-12")})
  ```
  Lsd. Greenwitch Mean Time: [https://greenwichmeantime.com/current-time/](https://greenwichmeantime.com/current-time/)
- Példa séma definiálásra (NoSQL-ben nem kötelező a séma használata)

  ```
  db.createCollection("students", {
   validator: {
      $jsonSchema: {
         bsonType: "object",
         required: [ "name", "year", "major", "address" ],
         properties: {
            name: {
               bsonType: "string",
               description: "must be a string and is required"
            },
            year: {
               bsonType: "int",
               minimum: 2017,
               maximum: 3017,
               description: "must be an integer in [ 2017, 3017 ] and is required"
            },
            major: {
               enum: [ "Math", "English", "Computer Science", "History", null ],
               description: "can only be one of the enum values and is required"
            },
            gpa: {
               bsonType: "double",
               description: "must be a double if the field exists"
            },
            address: {
               bsonType: "object",
               required: [ "city" ],
               properties: {
                  street: {
                     bsonType: "string",
                     description: "must be a string if the field exists"
                  },
                  city: {
                     bsonType: "string",
                     description: "must be a string and is required"
                  }
               }
            }
         }
      }
   }})

  Majd dokumentum beszúrása:
  db.students.insert({name: "Jedlik Ányos", year: NumberInt(2022), major: "Math", gpa: 3.14, address: {street: "Szent István út 7.", city: "Győr"}})
  ```

## 3.2 Olvasás (Read)

- Átváltás a "sample_mflix" adatbázisra

```
use sample_mflix
```

- Filmek listázása (nincs szűrés)

```
db.movies.find({})
```

- Szűrés egy feltétel szerint, reguláris kifejezéssel

```
db.movies.find({title: /^Best/i})
```

- És kapcsolat projekcióval és rendezéssel
> A $and operátor elhagyható, az "és" kapcsolat az alapértelmezés

```
db.movies.find({year: {$gt: 1995, $lte: 2000}, countries: "Hungary"}, {year: 1}).sort({year: -1})
```

- Vagy kapcsolat {$or[{feltétel_1}, {feltétel_2}, ...]}

```
db.movies.find({$or:[{countries: "Hungary"}, {countries: "Germany"}]}, {countries: 1})
db.movies.find({$or:[{title: /^seven/i}, {title: /eight$/i}]}, {title: 1})
```
## 3.3 Módosítás (Update)
- $set (beállító) operátor használata, egy dokumentum módosítása
```
db.movies.updateOne({_id: ObjectId("573a1393f29313caabcdc87b")}, {$set: {"imdb.rating": 1}})
```
- $mul (szorzás) operátor használata, több dokumentum módosítása
```
db.movies.updateMany({countries: "Hungary"}, {$mul: {"imdb.rating": 2}})
```
- $inc (increment: növelés, negatív is lehet a növekmény) operátor használata, több dokumentum módosítása
```
db.movies.updateMany({countries: "Hungary"}, {$inc: {"imdb.rating": -3}})
```

## 3.4 Törlés (Delete)
- Egy dokumentum törlése
```
db.movies.deleteOne({_id: ObjectId("573a1393f29313caabcdc87b")}),
```
- Több dokumentum törlése
```
db.movies.deleteMany({countries: "Romania"})
```
- Teljes kollekció eldobása
```
db.movies.drop()
```

# 4. MongoDB - Aggregation Pipeline
- Aggregation with the orders Data Set (in sample_examples databse)

> MongoDB Manual: [https://www.mongodb.com/docs/manual/core/aggregation-pipeline/](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/)

- Két tábla összekapcsolása: $lookup (mongoose: populate parancs is használható)
```
$lookup szintaxisa:
{
  $lookup:
   {
     from: <összekapcsolandó kollekció neve>,
     localField: <helyi kapcsolatot tartó mező> (PK vagy FK),
     foreignField: <távoli kapcsolatot tartó mező> (FK vagy PK),
     as: <kapcsolt mező azonosítója> (típus array)
   }
}
```

```
Példa: A movies és a comments kollekció összekapcsolása
("great" szóval kezdődik a címe és 3db kommentet kapott a film)
db.movies.aggregate([
    {$match: {title: /^great/i}},
    {$lookup: {
                from: "comments",
                localField: "_id",
                foreignField: "movie_id",
                as: "comments"
               }},
    {$match: {comments: {$size: 3}}},
    {$project: {title:1 , comments: 1}}
])
```
- Aggregation with the Zip Code Data Set (in sample_training databse)

> MongoDB Manual: [https://www.mongodb.com/docs/manual/tutorial/aggregation-zip-code-data-set/](https://www.mongodb.com/docs/manual/tutorial/aggregation-zip-code-data-set/)

