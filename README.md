
# pytest reports => elasticsearch

this provides a starting point for exploring the viability of sending pytest
test reports to elasticsearch so that kibana can be used to query, aggregate,
and visualize test data.

warning: none of this should be used in production without performing your own
due dilligence.

## approach

we're going to run [sqlalchemy's](https://github.com/sqlalchemy/sqlalchemy) unit tests to generate test data (the largest,
stablest, fastest pytest test suite i could find), use the [pytest-elk-reporter](https://github.com/fruch/pytest-elk-reporter)
plugin to upload those reports directly into a [dev ELK stack](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html), and use kibana to
explore and visualize the results.

## setup

1. clone this repo
1. cd into this repo's directory
1. double-check docker is configured for at least 4 GiB of memory ([ref](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html))
1. `docker-compose up`
1. run this command after a few seconds to be sure your dev elk cluster is up:

        curl -X GET "localhost:9200/_cat/nodes?v&pretty"

1. run this command so new indices can be created in your dev elk cluster automatically:

        curl -X PUT "localhost:9200/_cluster/settings" \
            -H 'Content-Type: application/json' \
            -d'
        {
            "persistent": {
                "action.auto_create_index": "true"
            }
        }
        '

1. clone sqlalchemy into some other directory:


        git clone git@github.com:sqlalchemy/sqlalchemy.git


1. `cd sqlalchemy`
8. apply this patch to your sqlalchemy clone (also included as `sqlalchemy-test-config.patch`):

        diff --git a/setup.cfg b/setup.cfg
        index fd196f4f5..3853d5e7a 100644
        --- a/setup.cfg
        +++ b/setup.cfg
        @@ -84,6 +84,8 @@ where = lib
        [tool:pytest]
        addopts = --tb native -v -r sfxX --maxfail=25 -p no:warnings -p no:logging
        python_files = test/*test_*.py
        +es_address = localhost:9200
        +es_index_name = test_data

        [upload]
        sign = 1
        diff --git a/tox.ini b/tox.ini
        index 966048877..c815c7280 100644
        --- a/tox.ini
        +++ b/tox.ini
        @@ -19,6 +19,7 @@ deps=
            pytest>=4.6.11,<5.0; python_version < '3'
            pytest>=6.2; python_version >= '3'
            pytest-xdist
        +     pytest-elk-reporter
            greenlet != 0.4.17
            mock; python_version < '3.3'
            importlib_metadata; python_version < '3.8'

1. depending on your python setup, you may need to tweak some things with this
   next command but run it as-is to see what's broken first:

        tox -e py39-sqlite

    if that worked and sqlalchemy's unit tests are running your computer (after
    ~1 minute to warm up) then you can stop reading. otherwise...

    if it says it can't find `tox`:

        pip3 install tox

    if it says it can't find `python3.9`:

        tox -e py38-sqlite

    you can't proceed unless you get the sqlalchemy unit tests running locally with
    those test config mods applied. we are using sqlalchemy to generate a ton of
    test data and we need this part to work in order for the next steps to be
    possible.

1. load up your dev kibana: http://localhost:5601

1. at this point you might be reading some message about adding your data, just
    ignore these and find a link that says something like "i think you have my
    data now" or something

1. it'll prompt you to create an index, just find the one that already exists:
    `test_data` and if you have to give a pattern put `test_data*`

1. don't bother getting fancy with the fields, find the hamburger menu in the upper left and find a link that says 'explore'

1. be sure you set the time filter to a range that should contain the test data from your sqlalchemy unit test run

1. behold: your test results are being uploaded to elasticsearch in real time and are aggregatable
