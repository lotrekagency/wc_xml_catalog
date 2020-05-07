# WC XML Catalog

This is a Docker microservice for generating XML catalogs from WooCommerce.

XML feeds files will be stored in `/app/feeds`

## Use it in Docker compose

    xmlcatalog:
        image: lotrekagency/wc_xml_catalog
        volumes:
            - ./xmlfeeds:/app/feeds
            - ./your_config.json:/app/config.json
        depends_on:
            - redis
        env_file: ./envs/prod/myxmlcatalog.env
        networks:
          - webnet


    redis:
        image: library/redis:latest
        restart: unless-stopped
        networks:
          - webnet

## Configuration
The service needs to be configured by setting up some environment variables for requests handling and files structure.
Then the output can be fully customized by filling in the json file with the tags you want.

### Service configuration
There are several environment variables you can set up.

    WOO_HOST=https://www.mywoocommerce.com/wp-json/wc/v3
    WOO_CONSUMER_KEY=ck_f4k3889898tgddfrr709eaade5e39a361c782
    WOO_CONSUMER_SECRET=cs_4g41nf4ke69797adsas23458af18253aad51a
    LANGUAGES=it,en
    XML_SITE_NAME=My Woo Commerce
    XML_SITE_HOST=www.mywoocommerce.com
    XML_TYPES_IN_PRODUCTS=variable
    XML_TYPES_IN_VARIATIONS=variation,simple

The `WOO_HOST`, `WOO_CONSUMER_KEY` and the `WOO_CONSUMER_SECRET` are the three main WooCommerce API parameters for products retrieving.<br>
`LANGUAGES` is related to the list of languages for each one the service will generate the products and the variations files. Languages are separated by a comma.<br>
`XML_SITE_NAME` is the name of your WooCommerce site.<br>
`XML_SITE_HOST` is the URL of your WooCommerce site.<br>
`XML_TYPES_IN_PRODUCTS` contains the list of product types the products file will contain. Each product type is separated by a comma.<br>
`XML_TYPES_IN_VARIATIONS` contains the list of product types the variations file will contain. Each product type is separated by a comma.

The product types values are: simple, grouped, virtual, downloadable, variable, variation, external.

### Feed configuration
The output feed can be customized by modifying the `config.json` file.
This file contains tags and values for the output of every product type indicated in the environment setup.

In the following example two types of products are configured: `variable` and `variation`.


    {
      "variable" : {
        "g:brand" : {
          "default" : "XML_SITE_NAME"
        },
        "g:id" : {
          "attribute" : "id"
        },
        "g:title" : {
          "attribute" : "name"
        },
        "g:condition": {
          "static" : "New"
        },
        "g:image_link" : {
          "attribute" : "images[0].src"
        },
        "g:availability" : {
          "attribute" : "in_stock",
          "replacer" : {
            "True" : "in stock",
            "False" : "out of stock"
          }
        }
    }
      },
      "variation" : {
        "g:title" : {
          "parent" : "meta_data(key=title).value",
          "attribute" : "attributes[0].option"
        }
        "g:shipping" : {
          "default" : "default_shippings",
          "unique" : true
        },
        "g:sale_price" : {
          "attribute" : "sale_price",
          "suffix" : " EUR"
        },
        "g:category" : {
          "attribute" : "categories.name",
          "separator" : " - "
        }
        "g:description" : {
          "attribute" : "meta_data(key=description).value"
        }
      }
    }

Each product type configuration contains objects for each tag in the output feed.

Example:

    "g:title" : {
      "attribute" : "name"
    }

With this configuration the output will be the following.

    <g:title>
      Your product name
    </g:title>

As you can see, the name of the objects is related to the tag name.
The tag object contains the output value paths.
The values can be retrieved in four ways:
* `static`, for static strings outputs
* `attribute`, for values retrieving from products fields
* `parent`, for values retrieving from parent products fields
* `default`, for values retrieving from default service variables


#### static
The `static` will output it's string value.

Example:

    "g:condition" : {
      "static" : "New"
    }

The output will be the following.

    <g:condition>
      New
    </g:condition>

This tag may also relate to a list of configuration objects: in this way it's possibile to configure a tag with sub-tags.

Example:

    "g:tax" : {
      "static" : [
        {
          "rate" : {
            "static" : "22.000"
          }
        },
        {
          "country" : {
            "static" : "IT"
          }
        }
      ],
    }

The output will be like the following.

    <g:tax>
      <rate>
        22.000
      </rate>
      <country>
        IT
      </country>
    </g:tax>

#### attribute

The `attribute` relates to the value retrieving by field path of the product object.
It's possibile to indicate the specific path of a value by indexing a list object or selecting it.

Example:

    "g:image_link" : {
      "attribute" : "images[0].src"
    }

It retrieves the `src` value from `images[0]` and the output will be like the following.

    <g:image_link>
      https://yourimage.png
    </g:image_link>

Another example with list selecting method:

    "g:description" : {
      "attribute" : "meta_data(key=description).value"
    }

It retrieves the `value` value from the `meta_data` object were `key` is equal to `description`.
The output will be like the following.

    <g:description>
      Your description
    </g:description>


If the penultimate field is a list, the service relates all the fields of every object of the list.

#### parent

The `parent` key works like `attribute` but refers to parent fields.
Parents are set automatically from the service: variable and simple products do not have parents, while variation products have their variable father product.

An example with this tag:

    "g:title" : {
      "parent" : "title"
    }

It retrieves the parent product `title` value, and the output will be the following.

    <g:title>
      Parent product description
    </g:title>

#### default

The `default` key relates to the service variables.
Available variables are:
* `XML_SITE_NAME`, contains the site name
* `XML_SITE_HOST`, contains the site host
* `XML_FEED_DESCRIPTION`, contains the site description
* `default_shippings`, contains the list of all the shippings retrieved
* `default_tax_rates`, contains the list of all the tax rates retrieved

Example:

    "g:brand" : {
      "default" : "XML_SITE_NAME"
    }

It takes the value of the attribute set up in the first configuration and the output will be like the following.

    <g:brand>
      Your brand name
    </g:brand>

#### Extra keys
There are some extra keys for strings formatting:
* `unique`, if it's `true`, every object of the list will be repeated with the father object name
* `suffix`, contains the suffix string
* `prefix`, contains the prefix string
* `separator`, contains the string which will separate the values of every object list
* `replacer`, contains a dictionary with the values that need to be replaced in the output feed
* `fatal`, if it's `true` and the retrieved value it's `None`, the object will not loaded on the output feed


The setup of these two files let users to generate customized feeds for Google Merchant and Facebook Catalogs.

## Test it

    cd tests/
    pytest .
