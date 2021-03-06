# Bulk Insert: 5 milioni di record in meno di 10 secondi

**Keywords:** Bulk Insert, Massive Insert, Database, Laravel Query
Builder, Laravel, Optimization, Performance

## Introduzione
Quante volte vi è capitato di dover caricare su database dei dati provenienti da un file csv fornito dal cliente? oppure 
di fare il backup ed il restore dei dati da una versione di un sito ad un altro (e quindi senza la possibilità di usare un dump del database)? 

L'algoritmo classico che viene utilizzato in questi casi è il seguente:

* carichi il file sul server
* lo apri
* fai un ciclo che legge riga per riga
* ad ogni riga fai l'insert su database

Queste semplici istruzioni però, come molti di voi avranno sperimentato sulla loro pelle, al crescere delle
righe da inserire, cresce anche il tempo necessario, fino ad arrivare anche a diverse ore. Ma è veramente questo l'unico modo per 
eseguire questo genere di operazioni? non esiste qualcosa di più efficiente?

La risposta è si, esiste. Però la soluzione non ve la darò subito. Vi condurrò invece attraverso un percorso
che partirà dalla situazione meno efficiente fino ad arrivare ad una situazione ottimale. Proverò quindi a 
farvi comprendere non solo il come, ma anche il perchè.

Se però siete impazienti e volete avere subito la soluzione, cliccate qui!

```
NOTA: Nonostante l'articolo sia relativo al mondo Laravel, le conclusione a cui si arriva sono facilmente replicabili
anche su altri framework, una volta compresi i principi.
```

Eloquent è uno strumento potentissimo.
Permette la scrittura di query anche molto complesse con poche righe di
codice, aumentando anche la leggibilità del codice, rendendolo più
semantico ed auto documentante.

Prendiamo ad esempio un'ipotetica applicazione di gestione di
biblioteche.
Uno dei servizi disponibili è sicuramente quello di elenco degli autori
e dei rispettivi libri scritti. Mi aspetto quindi di avere un servizio
REST che mi ritorni un json all'incirca così (in versione molto
semplificata).

```json
[
   {
      "id": 42,
      "name": "Asimov Isaac",
      "nationality": "US",
      "books": [
         {
            "id": 456,
            "title": "Foundation",
            "year": "1951"
         },
         {
            "id": 4327,
            "title": "Foundation and Empire",
            "year": "1952"
         }
      ]
   },
   { "id": "..." }
]
```

In Laravel questo risultato lo otterrei così.

```php
/** MODEL **/

Author extends Model {
   Protected $table = 'authors';

   public function scopeStartingWith($query, $name) {
      return $query->where('name', 'LIKE', $name);
   }

   public function scopeBornIn($query, $nationality) {
      return $query->where('nationality', $nationality);
   }

   public function books()
   {
      return $this->hasMany('App/Book');
   }
}

/** CONTROLLER **/

// Elenco degli autori americani che iniziano con 'Asim' e dei loro libri
$authors = Author::startingWith('Asim')->bornIn('US')->with('books');

return response()->json($authors, 200);
```

Mentre lo stesso risultato, utilizzando PDO invece che Eloquent, lo si
otterrebbe con questo codice:

```php
$db = /** Codice per ottenere l'istanza di PDO **/

$sql = 'SELECT A.*, B.id as book_id, B.title as book_title, B.year as book_year
        FROM authors A
        WHERE A.name LIKE :name AND
              A.nationality = :nationality
        JOIN books B ON B.author_id = A.id';

$statement = $db->prepare($sql);

$statement->execute([':name' => 'Asim%', ':nationality' => 'US']);


$authors = [];

while ($row = $sth->fetch(PDO::FETCH_ASSOC)) {
    $authorId = $row['id'];

    // Aggrego le informazioni dell'autore
    if (!isset($authors[$authorId])) {
        $authors[$authorId] = [
            'id'          => $authorId,
            'name'        => $row['name'],
            'nationality' => $row['nationality'],
            'books'       => []
        ];
    }

    // Aggiungo le informazioni sui libri
    $authors[$authorId]['books'][] = [
        'id'    => $row['book_id'],
        'title' => $row['book_title'],
        'year'  => $row['book_year']
    ];
}

return response()->json($authors, 200);
```

Eloquent quindi ci permette di astrarre meglio, di scrivere codice più
chiaro e manutenibile, nascondendo però molto codice sotto il "cofano".

Laravel viene definito un framework general purpose nel senso che
cerca di offire agli sviluppatori i tool necessari per risolvere le
problematiche più comuni in maniera rapida. Diaciamo quindi che l'utilizzo
standard di Laravel potrebbe andare bene per il 90% delle volte
(dipende ovviamente dalla tipologia di progetto). Inoltre, tutte queste
commodities hanno un costo in termini di performance che nei casi più
comuni sono trascurabili ma che in altri, come quello di cui parla questo
articolo, non sono assolutamente accettabili.

Esistono quindi tutta una serie di casi limite che necessitano maggior studio ed
approfondimento su come funziona il framework al suo interno, e sulle
tecnologie su cui si basa.

`Know your tools!`

Generalmente i casi limite sono quelli che hanno a che fare con i grandi
numeri. Ogni qualvolta si hanno a che fare con quantità massive, la
nostra architettura viene messa a dura prova, ed è proprio lì che si
nota la differenza tra un professionista ed uno smanettone.


In questo articoli quindi ci concentreremo su un caso specifico:
le insert massive su db, e per massive intendo milioni di righe.

## I benchmark
Partiremo da esempi semplici ma significativi. Faremo un percorso che
partirà da un caso di utilizzo di Laravel standard fino ad arrivare, ottimizzazione
dopo ottimizzazione, ad ottenere un codice estremamente efficiente. I test
si svolgeranno attraverso l'utilizzo di un benchmark appositamente creato.
Di seguito le istruzioni per poterlo installare, nel caso voleste a vostra volta
eseguire i test.

```bash
git clone https://github.com/cnastasi/laraver-query-benchmark.git
composer install
php artisan key:generate
cp .env.example .env
# Configurate .env con i dati di un vostro DB
php artisan migrate
```

I test, sotto forma di comando artisan, sono tutti disponibili sotto la
cartella `app/Console/Commands`.

Ogni classe di test implementa la seguente interfaccia: 

```php
interface BulkInsertTest {
    public function bulkInsert(Callable $lineFactory):void;
}
```

Dato che la lettura del file csv influisce in percentuale minima, passeremo alla funzione una funzione 
anonima, 

### Esempio 1: Eloquent base
Come primo esempio utilizzeremo quello che ci 
Leggendo la documentazione di Laravel, uno dei modi per poter salvare un
dato su DB è il seguente:

```php
function bulkInsert(
    $user = new User();

    $user->name = $item['name'];
    $user->email = $item['email'];
    /** other assignments */

    $user->save();
```

Questo è anche il modo che viene preferito dalla maggior parte
degli sviluppatori Laravel.

Proviamo quindi a capire quanto questo codice sia efficiente.

```bash
php artisan benchmark:eloquent1 1
php artisan benchmark:eloquent1 10
php artisan benchmark:eloquent1 100
php artisan benchmark:eloquent1 1000
php artisan benchmark:eloquent1 10000
```
Con il relativo risultato.

| Execution (sec)| # Samples | sec x sample  | MB Usage | MB Peak |
| :------------: | :-------: | :-----------: | :------: | :-----: |
| 0.011          | 1         | 11.233 x10^-3 | 9.64     | 9.91    |
| 0.035          | 10        | 3.543 x10^-3  | 9.64     | 9.91    |
| 0.309          | 100       | 3.092 x10^-3  | 9.64     | 9.91    |
| 3.068          | 1K        | 3.068 x10^-3  | 9.64     | 9.91    |
| 30.937         | 10K       | 3.094 x10^-3  | 9.64     | 9.91    |


L'andamento è lineare, con un consumo di memoria apparentemente costante.
Mi aspetterei quindi che con 1 milione di record il tempo di esecuzione
duri circa 3000 secondi (pari a 50 minuti) e, di conseguenza, con 10
milioni di record, arrivare addirittura a 8 ore e 20 minuti. Un tempo
decisamente non accettabile. Capiamo quindi se ed in che modo possiamo
ottimizzare.

## Esempio 2: Eloquent - 
