!SLIDE

# MongoDB + Solr
## ETL et indexation full-text

<br/>
<br/>
<br/>
<h3><span class="no_em">Retour d'expérience sur</span> <span class="t_h">Hacker</span><span class="t_b">Books</span><span class="no_em">.com</span></h3>
<h3><span class="no_em">Thibaut Barrère @ LoGeek.fr</span></h3>

!SLIDE

# Ze Plan

<div style="margin-left: 250px;">
  <h2 style="text-align: left;">1 - Fonctionnement du site</h2>
  <h2 style="text-align: left;">2 - ETL / MongoDB</h2>
  <h2 style="text-align: left;">3 - Indexation / Solr</h2>
  <h2 style="text-align: left;">4 - Pitfalls</h2>
</div>

!SLIDE full-page-image

![""](hackerbooks-home.png)

!SLIDE full-page-image

## Stack Overflow

![""](so.png)

!SLIDE full-page-image

## Hacker News

![""](hn.png)

!SLIDE full-page-image

![""](hackerbooks-list.png)

!SLIDE full-page-image

![""](hackerbooks-detail.png)

!SLIDE

<h1>E<span class='no_em'>xtract</span> T<span class='no_em'>ransform</span> L<span class='no_em'>oad</span></h1>

<div style="text-align: center;">
  <img src="gears.png" style="width: 20%;">
</div>

!SLIDE

# Sources de données brutes

<br/>
## Dump Stack Overflow (3 Gb XML)
<br/>
## Dump Hacker News (1 Gb XML)
## + Crawler Hacker News (updates)

!SLIDE

## I Can Haz Diagram?

![""](diagram.png)

!SLIDE

## Extraction/conformation

    @@@ruby
    HackerNews::SaxReader.new(file).each do |record|
      result = {}      
      result['discussion_id'] = record['ParentID']
      result['points'] = record['Points']
      # ...
      
      batch << result
      if batch.size > 1000
        collection.insert(batch, :safe => true)
        batch = []
      end
    end

### => API pratique pour mapping / bulk load / lookup / truncate

!SLIDE

## Upsert (update or insert)

    @@@ruby
    Quotes.collection.update(
      quote.slice('asin', 'site', 'id'), # clé composite
      { '$set' => quote },               # nouvelle valeur  
      { :upsert => true }
    )

### => code identique pour:
### - une consolidation complète
### - une consolidation incrémentale sur un jour
### - ou sur un seul élément

!SLIDE

### Avantages de MongoDB en ETL

## 1) Absence de schéma

!SLIDE

### Avantages de MongoDB en ETL

## 2) Richesse de requêtage

!SLIDE

### Avantages de MongoDB en ETL

## 3) Upsert

!SLIDE

### Avantages de MongoDB en ETL

## 4) Manipulation des tables/databases

!SLIDE

### Avantages de MongoDB en ETL

## 5) Scalabilité/sharding

!SLIDE

### Friction lors d'un travail de type ETL

<h1>MongoDB <span class='no_em'>est à</span> SQL</h1>

<h2><span class='no_em'>ce que</span> Ruby <span class='no_em'>est à</span> Java</h2>

!SLIDE

# Indexation full-text

<div style="text-align: center;">
  <img src="searchmini.png">
</div>

!SLIDE

# Possibilités envisagées

## MongoDB seul
## Sphinx
## Solr

!SLIDE smaller_code

## Adapteur MongoMatic/Solr (1/2)

    @@@ruby
    class InstanceAdapter < Sunspot::Adapters::InstanceAdapter
      def id
        @instance['_id']
      end
    end

    Sunspot::Adapters::InstanceAdapter.register(InstanceAdapter, Book)

!SLIDE smaller_code

## Adapteur MongoMatic/Solr (2/2)

    @@@ruby
    class DataAccessor < Sunspot::Adapters::DataAccessor
      def load(id)
        @clazz.find_one(BSON::ObjectId(id))
      end

      def load_all(ids)
        ids = ids.map { |_id| BSON::ObjectId(_id) }
        @clazz.find('_id' => { '$in' => ids}).to_a
      end
    end

    Sunspot::Adapters::DataAccessor.register(DataAccessor, Book)

!SLIDE

## Définition des indexes

    @@@ruby
    Sunspot.setup(Book) do

      text :asin, :title, :description, :stored => true

      string :quoted_by, :multiple => true
      string :quoted_on, :multiple => true

      integer :karma
      boolean :kindle_edition

    end

!SLIDE smaller_code

## Accesseurs de données (1/2)

    @@@ruby
    def karma
      Quote.collection.find('asin' => asin).count
    end

### => 27

    @@@ruby
    def kindle_edition
      amazon_versions_xml.get_array('binding').grep(/kindle/i).size > 0
    end

### => true

!SLIDE smaller_code

## Accesseurs de données (2/2)

    @@@ruby
    def quoted_on
      
      values = Quote.collection.find(
        { 'asin' => asin }, 
        { :fields => { 'site' => 1, '_id' => 0 }})
      
      values.map { |e| e['site'] }.uniq.sort
    
    end

### => ['HN', 'SO']

!SLIDE

## Recherche simple

    @@@ruby
    Sunspot.search(Book) do
      keywords 'SQL', :fields => [:title, :description]
      
      order_by :score, :desc
      order_by :karma, :desc
    end

!SLIDE

## Recherche tunée

    @@@ruby
    Sunspot.search(Book) do
      keywords "SQL -Ruby" do
        boost_fields :title => 1.5, :description => 0.1
        boost(function { product(:karma, 3) }) 
      end
      
      with :quoted_on,'HN'
      with :kindle_edition,true
      
      paginate :page => 3, :per_page => 10
    end
