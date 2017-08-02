# Bulk Insert: 5 milioni di record in meno di 10 secondi

**Keywords:** Bulk Insert, Massive Insert, Database, Laravel Query
Builder, Laravel, Optimization, Performance

Eloquent è uno strumento potentissimo.
Permette la scrittura di query anche molto complesse con poche righe di
codice, aumentando anche la leggibilità del codice, rendendolo più
semantico ed auto documentante.

Prendiamo ad esempio un’ipotetica applicazione di gestione di
biblioteche.
Uno dei servizi disponibili è sicuramente quello di elenco degli autori
e dei rispettivi libri scritti. Mi aspetto quindi di avere un servizio
REST che mi ritorni un json all’incirca così (in versione molto
semplificata).

```php
[
   {
      “id”: 42,
      “name”: “Asimov Isaac”,
      “nationality”: “US”,
      “books”: [
         {
            “id”: 456,
            “title”: “Foundation”,
            “year”: “1951”
         },
         {
            “id”: 4327,
            “title”: “Foundation and Empire”,
            “year”: “1952”
         }
      ]
   },
   { … }
]
```

In Laravel questo risultato lo otterrei così.

```php
/** MODEL **/

Author extends Model {
   Protected $table = ‘authors’;

   public function scopeStartingWith($query, $name) {
      return $query->where(‘name’, ‘LIKE’, $name);
   }

   public function scopeBornIn($query, $nationality) {
      return $query->where(‘nationality’, $nationality);
   }

   public function books()
   {
      return $this->hasMany('App/Book');
   }
}

/** CONTROLLER **/

// Elenco degli autori americani che iniziano con ‘Asim’ e dei loro libri
$authors = Author::startingWith(‘Asim’)->bornIn(‘US’)->with(‘books’);

return response()->json($authors, 200);
```

Mentre lo stesso risultato, utilizzando PDO invece che Eloquent, lo si
otterrebbe con questo codice:

```php
$db = /** Codice per ottenere l’istanza di PDO **/

$sql = 'SELECT A.*, B.id as book_id, B.title as book_title, B.year as book_year
        FROM authors A
        WHERE A.name LIKE :name AND
              A.nationality = :nationality
        JOIN books B ON B.author_id = A.id';

$statement = $db->prepare($sql);

$statement->execute([':name' => ’Asim%’, ':nationality' => 'US']);


$authors = [];

while ($row = $sth->fetch(PDO::FETCH_ASSOC)) {
    $authorId = $row[‘id’];

    // Aggrego le informazioni dell’autore
    if (!isset($authors[$authorId])) {
        $authors[$authorId] = [
            ‘id’          => $authorId,
            ‘name’        => $row[‘name’],
            ‘nationality’ => $row[‘nationality’],
            ‘books’       => []
        ];
    }

    // Aggiungo le informazioni sui libri
    $authors[$authorId][‘books’][] = [
        ‘id’    => $row[‘book_id’],
        ‘title’ => $row[‘book_title’],
        ‘year’  => $row[‘book_year’]
    ];
}

return response()->json($authors, 200);
```

Eloquent quindi ci permette di astrarre meglio, di scrivere codice più
chiaro e manutenibile, nascondendo però molto codice sotto il “cofano”.

Queste commodities tuttavia hanno un costo in termini di performance.
Laravel viene definito un framework general purpose, il che significa
che cerca di risolvere le problematiche più comuni.

L’utilizzo base di Laravel quindi va bene nel 90% delle volte.

Esistono però una serie di casi limite che necessitano maggior studio ed
approfondimento su come funziona il framework al suo interno.
In questi casi limite forse l’utilizzo della soluzione generica potrebbe
non essere più efficiente ed adatta. Generalmente i casi limite sono
quelli che hanno a che fare con i grandi numeri. Questo proprio perchè i
framework offrono tanta magia ad un costo computazionale elevato.
L’utilizzo di questa magia può essere accettabili in operazioni atomiche
mentre incomincia a diventare assolutamente inefficiente l’utilizzo in
casi con migliaia o anche milioni di operazioni. In questo articoli ci
concentreremo su un caso specifico: le insert massive su db.
