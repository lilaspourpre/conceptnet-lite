# conceptnet-lite

Conceptnet-lite is a Python library for working with ConceptNet offline without the need for PostgreSQL.

The library comes with Apache License 2.0, and is separate from ConceptNet itself. The ConceptNet is available under [CC-BY-SA-4.0](https://creativecommons.org/licenses/by-sa/4.0/) license, which also applies to the formatted database file that we provide. See [here](https://github.com/commonsense/conceptnet5/wiki/Copying-and-sharing-ConceptNet) for the list of conditions for using ConceptNet data.

This is the official citation for ConceptNet if you use it in research:

> Robyn Speer, Joshua Chin, and Catherine Havasi. 2017. "ConceptNet 5.5: An Open Multilingual Graph of General Knowledge." In proceedings of AAAI 31.

The basic usage of `conceptnet-lite` library is as follows.

## Downloading the ConceptNet database 

ConceptNet releases happen once a year. You can use `conceptnet-lite` to build your own database from the raw assertions file, but if there is a pre-built file it will be faster to just get that one. `conceptnet-lite` can download it automatically if `db_download_url` argument is specified.

A link to a compressed database for ConceptNet 5.7 will be added shortly. 

```python
import conceptnet_lite

conceptnet_lite.connect(db_path='/path/to/conceptnet', db_download_url="http://some/url.zip")
```

## Building the database for a new release.

If a database file is not found in the folder specified in the `db_path` argument, `conceptnet-lite` will attempt to automatically download the raw assertions file from [here](https://github.com/commonsense/conceptnet5/wiki/Downloads) and build the database. This takes several hours, so we recommend getting the pre-built file.

```python
import conceptnet_lite

conceptnet_lite.connect(db_path='/path/to/future/database/')
```

## Loading the ConceptNet database 

Once you have the database file, all you need to do is pass either the full path to it or a path to the folder containing a "conceptnet.db" file to the `db_path` argument:

```python
import conceptnet_lite

conceptnet_lite.connect(db_path='/path/to/conceptnet.db')
```


## Accessing concepts

Concepts objects are created by looking for every entry that matches the input string exactly.
If none is found, the `peewee.DoesNotExist` exception will be raised.

```python
from conceptnet_lite import Label

cat_concepts = Label.get(text='cat').concepts  #
for c in cat_concepts:
    print("    Concept URI:", c.uri)
    print("    Concept text:", c.text)
```
```console
Concept URI: /c/en/cat
Concept text: cat
Concept URI: /c/en/cat/n
Concept text: cat
Concept URI: /c/en/cat/n/wn/animal
Concept text: cat
Concept URI: /c/en/cat/n/wn/person
...
```

`concept.uri` provides access to ConceptNet URIs, as described [here](https://github.com/commonsense/conceptnet5/wiki/URI-hierarchy). You can also retrieve only the text of the entry by `concept.text`.

## Working with languages

You can limit the languages to search for matches. Label.get() takes an optional `language` attribute that is expected to be an instance `Language`, which in turn is created by calling `Language.get()` with `name` argument.
List of available languages and their codes are described [here](https://github.com/commonsense/conceptnet5/wiki/Languages).

```python
from conceptnet_lite import Label, Language

english = Language.get(name='en')
cat_concepts = Label.get(text='cat', language=english).concepts  #
for c in cat_concepts:
    print("    Concept URI:", c.uri)
    print("    Concept text:", c.text)
    print("    Concept language:", c.language.name)
```

```console
    Concept URI: /c/en/cat
    Concept text: cat
    Concept language: en
    Concept URI: /c/en/cat/n
    Concept text: cat
    Concept language: en
    Concept URI: /c/en/cat/n/wn/animal
    Concept text: cat
    Concept language: en
    Concept URI: /c/en/cat/n/wn/person
    Concept text: cat
    Concept language: en
...
```


## Querying edges between concepts

To retrieve the set of relations between two concepts, you need to create the concept objects (optionally specifying the language as described above). `cn.edges_between()` method retrieves all edges between the specified concepts. You can access its URI and a number of attributes, as shown below.

Some ConceptNet relations are symmetrical: for example, the antonymy between *white* and *black* works both ways. Some relations are asymmetrical: e.g. the relation between *cat* and *mammal* is either hyponymy or hyperonymy, depending on the direction. The `two_way` argument lets you choose whether the query should be symmetrical or not.

```python
from conceptnet_lite import Label, Language, edges_between

english = Language.get(name='en')
introvert_concepts = Label.get(text='introvert', language=english).concepts
extrovert_concepts = Label.get(text='extrovert', language=english).concepts
for e in edges_between(introvert_concepts, extrovert_concepts, two_way=False):
    print("  Edge URI:", e.uri)
    print("  Edge name:", e.relation.name)
    print("  Edge start node:", e.start.text)
    print("  Edge end node:", e.end.text)
    print("  Edge metadata:", e.etc)
```
```console
  Edge URI: /a/[/r/antonym/,/c/en/introvert/n/,/c/en/extrovert/]
  Edge name: antonym
  Edge start node: introvert
  Edge end node: extrovert
  Edge metadata: {'dataset': '/d/wiktionary/en', 'license': 'cc:by-sa/4.0', 'sources': [{'contributor': '/s/resource/wiktionary/en', 'process': '/s/process/wikiparsec/2'}, {'contributor': '/s/resource/wiktionary/fr', 'process': '/s/process/wikiparsec/2'}], 'weight': 2.0}
```

* **e.relation.name**: the name of ConceptNet relation. Full list [here](https://github.com/commonsense/conceptnet5/wiki/Relations).

* **e.start.text, e.end.text**: the source and the target concepts in the edge

* **e.etc**: the ConceptNet [metadata](https://github.com/commonsense/conceptnet5/wiki/Edges) dictionary contains the source dataset, sources, weight, and license. For example, the introvert:extrovert edge for English contains the following metadata:

```json
{
	"dataset": "/d/wiktionary/en",
	"license": "cc:by-sa/4.0",
	"sources": [{
		"contributor": "/s/resource/wiktionary/en",
		"process": "/s/process/wikiparsec/2"
	}, {
		"contributor": "/s/resource/wiktionary/fr",
		"process": "/s/process/wikiparsec/2"
	}],
	"weight": 2.0
}
```

## Accessing all relations for a given concepts

You can also retrieve all relations between a given concepts and all other concepts, with the same options as above:

```python
from conceptnet_lite import Label, Language, edges_for

english = Language.get(name='en')
for e in edges_for(Label.get(text='introvert', language=english).concepts, same_language=True):
    print(e.start.text, "::", e.end.text, "|", e.relation.name)
```
```console
extrovert :: introvert | antonym
introvert :: extrovert | antonym
outrovert :: introvert | antonym
reflection :: introvert | at_location
introverse :: introvert | derived_from
introversible :: introvert | derived_from
introversion :: introvert | derived_from
introversion :: introvert | derived_from
introversive :: introvert | derived_from
introverted :: introvert | derived_from
...
```

The same set of edge attributes are available for `edges_between` and `edges_for` (e.uri, e.relation.name, e.start.text, e.end.text, e.etc).

Note that we have used optional argument `same_language=True`. By supplying this argument we make `edges_for` return
relations, both ends of which are in the same language. If this argument is skipped it is possible to get edges to
concepts in languages other than the source concepts language. For example, the same command as above with `same_language=False` will include the following in the output:

```console
kääntyä_sisäänpäin :: introvert | synonym
sulkeutua :: introvert | synonym
sulkeutunut :: introvert | synonym
introverti :: introvert | synonym
asociale :: introvert | synonym
introverso :: introvert | synonym
introvertito :: introvert | synonym
内向 :: introvert | synonym
```

## Accessing concept edges with a given relation direction

You can also query the relations that have a specific concept as target or source. This is achieved with `concept.edges_out` and `concept.edges_in`, as follows:

```python
from conceptnet_lite import Language, Label

english = Language.get(name='en')
concepts = Label.get(text='introvert', language=english).concepts  #
for c in concepts:
    print("    Concept text:", c.text)
    if c.edges_out:
        print("      Edges out:")
        for e in c.edges_out:
            print("        Edge URI:", e.uri)
            print("        Relation:", e.relation.name)
            print("        End:", e.end.text)
    if c.edges_in:
        print("      Edges in:")
        for e in c.edges_in:
            print("        Edge URI:", e.uri)
            print("        Relation:", e.relation.name)
            print("        End:", e.end.text)
```
```console
    Concept text: introvert
      Edges out:
        Edge URI: /a/[/r/etymologically_derived_from/,/c/en/introvert/,/c/la/introvertere/]
        Relation: etymologically_derived_from
        End: introvertere
...
      Edges in:
        Edge URI: /a/[/r/antonym/,/c/cs/extrovert/n/,/c/en/introvert/]
        Relation: antonym
        End: introvert
...
```

# Traversing all the data for a language

You can go over all concepts for a given language. For illustration, let us try Avestan, a "small" language with the code "ae" and vocab size of 371, according to the [ConceptNet language statistics](https://github.com/commonsense/conceptnet5/wiki/Languages).

```python
from conceptnet_lite import Language

mylanguage = Language.get(name='ae')
for l in mylanguage.labels:
    print("  Label:", l.text)
    for c in l.concepts:
        print("    Concept URI:", c.uri)
        if c.edges_out:
            print("      Edges out:")
            for e in c.edges_out:
                print("        Edge URI:", e.uri)
        if c.edges_in:
            print("      Edges in:")
            for e in c.edges_in:
                print("        Edge URI:", e.uri)
```
```console
  Label: 𐬨𐬀𐬰𐬛𐬀𐬌𐬌𐬀𐬯𐬥𐬀
    Concept URI: /c/ae/𐬨𐬀𐬰𐬛𐬀𐬌𐬌𐬀𐬯𐬥𐬀/n
      Edges out:
        Edge URI: /a/[/r/antonym/,/c/ae/𐬨𐬀𐬰𐬛𐬀𐬌𐬌𐬀𐬯𐬥𐬀/n/,/c/ae/𐬛𐬀𐬉𐬎𐬎𐬀𐬌𐬌𐬀𐬯𐬥𐬀/]
    Concept URI: /c/ae/𐬨𐬀𐬰𐬛𐬀𐬌𐬌𐬀𐬯𐬥𐬀
      Edges out:
        Edge URI: /a/[/r/external_url/,/c/ae/𐬨𐬀𐬰𐬛𐬀𐬌𐬌𐬀𐬯𐬥𐬀/,/c/en.wiktionary.org/wiki/𐬨𐬀𐬰𐬛𐬀𐬌𐬌𐬀𐬯𐬥𐬀/]
      Edges in:
        Edge URI: /a/[/r/etymologically_related_to/,/c/fa/مزدیسنا/,/c/ae/𐬨𐬀𐬰𐬛𐬀𐬌𐬌𐬀𐬯𐬥𐬀/]
        Edge URI: /a/[/r/etymologically_related_to/,/c/pal/𐭬𐭦𐭣𐭩𐭮𐭭/,/c/ae/𐬨𐬀𐬰𐬛𐬀𐬌𐬌𐬀𐬯𐬥𐬀/]
        Edge URI: /a/[/r/etymologically_related_to/,/c/xpr/𐭌𐭆𐭃𐭉𐭆𐭍/,/c/ae/𐬨𐬀𐬰𐬛𐬀𐬌𐬌𐬀𐬯𐬥𐬀/]
...
```


Todo:

- [ ] add database file link
