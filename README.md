# Searchable DataObjects

Searchable DataObjects is a module that permit to include DataObjects into frontend search.

## Introduction

Pages are not always the better way to implement things. For example site news can grow rapidly and the first side effect
would be a big and difficult to manage SiteTree. DataObjects help maintaining things clean and straight, but unfortunately
they are not included in frontend search. This module let you insert DataObject in search.

## Requirements

 * SilverStripe 3.2.*
 * zirak/htmlpurifier

### Installation

Install the module through [composer](http://getcomposer.org):

    composer require zirak/searchable-dataobjects
    composer update

Make the DataObject (or Pages) implement Searchable interface (you need to implement Link(), getSearchFilter(), getTitleFields(),
getContentFields()):

*Note: `getSearchFilterByCallback()` is an optional filter. If you don't plan on calculating any value to determine a returned `true` or `false` value it is suggested you don't add this function to your `DataObject` or `Page` type.*

```php
class DoNews extends DataObject implements Searchable {

    private static $db = array(
        'Title' => 'Varchar',
        'Subtitle' => 'Varchar',
        'News' => 'HTMLText',
        'Date' => 'Date',
    );
    private static $has_one = array(
        'Page' => 'PghNews'
    );

    /**
     * Link to this DO
     * @return string
     */
    public function Link() {
        return $this->Page()->Link() . 'read/' . $this->ID;
    }

    /**
     * Filter array
     * eg. array('Disabled' => 0);
     * @return array
     */
    public static function getSearchFilter() {
        return array();
    }

    /**
     * FilterAny array (optional)
     * eg. array('Disabled' => 0, 'Override' => 1);
     * @return array
     */
    public static function getSearchFilterAny() {
        return array();
    }

    /**
     * FilterByCallback function (optional)
     * eg. function($object){
     *  return ($object->StartDate > date('Y-m-d') || $object->isStillRecurring());
     * };
     * @return array
     */
    public static function getSearchFilterByCallback() {
        return function($object){ return true; };
    }

    /**
     * Fields that compose the Title
     * eg. array('Title', 'Subtitle');
     * @return array
     */
    public function getTitleFields() {
        return array('Title');
    }

    /**
     * Fields that compose the Content
     * eg. array('Teaser', 'Content');
     * @return array
     */
    public function getContentFields() {
        return array('Subtitle', 'Content');
    }
}
```

Here you are a sample page holder, needed to implement the Link() function into the DataObject:

```php
class PghNews extends Page {

    private static $has_many = array(
        'News' => 'DoNews'
    );

    public function getCMSFields() {
        $fields = parent::getCMSFields();

        /* News */
        $gridFieldConfig = GridFieldConfig_RelationEditor::create(100);
        // Remove unlink
        $gridFieldConfig->removeComponentsByType('GridFieldDeleteAction');
        // Add delete
        $gridFieldConfig->addComponents(new GridFieldDeleteAction());
        // Remove autocompleter
        $gridFieldConfig->removeComponentsByType('GridFieldAddExistingAutocompleter');
        $field = new GridField('Faq', 'Faq', $this->News(), $gridFieldConfig);
        $fields->addFieldToTab('Root.News', $field);

        return $fields;
    }
}

class PghNews_Controller extends Page_Controller {

    private static $allowed_actions = array(
        'read'
    );

    public function read(SS_HTTPRequest $request) {
        $arguments = $request->allParams();
        $id = $arguments['ID'];

        // Identifico la faq dall'ID
        $Object = DataObject::get_by_id('DoNews', $id);

        if ($Object) {
            //Popolo l'array con il DataObject da visualizzare
            $Data = array($Object->class => $Object);
            $this->data()->Title = $Object->Title;

            $retVal = $this->Customise($Data);
            return $retVal;
        } else {
            //Not found
            return $this->httpError(404, 'Not found');
        }
    }
}
```

Extend Page and the desired DataObjects through the following yaml:

```YAML
Page:
  extensions:
    - SearchableDataObject
DoNews:
  extensions:
    - SearchableDataObject
```

Run a `dev/build` and then populate the search table running PopulateSearch task:

    sake dev/build "flush=all"
    sake dev/tasks/PopulateSearch

Enjoy the news into the search results :)

### Modifying

#### Set the number of search results per page

Setting the `CustomSearch.items_per_page` config setting you can define, how many search results per page are shown. Default is 10

By default the search result is shown at the same page, so if you're searching e.g. on the */about-us/*, the results are
shown on */about-us/SearchForm/?s=foo*. If you don't like that, you can define any Page or Controller class in the
`CustomSearch.search_controller` setting. If you set this setting to `this`, the current page will be used. Defaults to `SearchPage`
and falls back to the current page if no SearchPage is found.

```YAML
CustomSearch:
  items_per_page: 15
  search_controller: SearchPage #page type to show the search
```

### TODO

 * Add other search method in configuration

### Suggested modules

 * Linkable DataObjects: http://addons.silverstripe.org/add-ons/zirak/linkable-dataobjects
