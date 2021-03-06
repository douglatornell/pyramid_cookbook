CouchDB and Pyramid
====================

If you want to use CouchDB (via the
`couchdbkit package <http://pypi.python.org/pypi/couchdbkit>`_)
in Pyramid, you can use the following pattern to make your CouchDB database
available as a request attribute. This example uses the starter scaffold.
(This follows the same pattern as the :doc:`mongodb` example.)

First add configuration values to your ``development.ini`` file, including your
CouchDB URI and a database name (the CouchDB database name, can be anything).

.. code-block:: ini
   :linenos:

    [app:main]
    # ... other settings ...
    couchdb.uri = http://localhost:5984/
    couchdb.db = mydb

Then in your ``__init__.py``, set things up such that the database is
attached to each new request:

.. code-block:: python
   :linenos:

    from pyramid.config import Configurator
    from pyramid.events import subscriber, NewRequest

    from couchdbkit import *

    @subscriber(NewRequest)
    def add_couchdb_to_request(event):
        request = event.request
        settings = request.registry.settings
        db = settings['couchdb.server'].get_or_create_db(settings['couchdb.db'])
        event.request.db = db


    def main(global_config, \**settings):
        """ This function returns a Pyramid WSGI application.
        """
        config = Configurator(settings=settings)
        """ Register server instance globally
        """
        config.registry.settings['couchdb.server'] = Server(uri=settings['couchdb.uri'])
        config.add_static_view('static', 'static', cache_max_age=3600)
        config.add_route('home', '/')
        config.scan()
        return config.make_wsgi_app()


At this point, in view code, you can use request.db as the CouchDB database
connection.  For example:

.. code-block:: python
   :linenos:

    from pyramid.view import view_config

    @view_config(route_name='home', renderer='templates/mytemplate.pt')
    def my_view(request):
        """ Get info for server
        """
        return {
            'project': 'pyramid_couchdb_example',
            'info': request.db.info()
        }

Add info to home template:

.. code-block:: html
   :linenos:

    <p>${info}</p>

CouchDB Views
-------------

First let's create a view for our page data in CouchDB. We will use the
ApplicationCreated event and make sure our view containing our page data.
For more information on views in CouchDB see
`Introduction to CouchDB views <http://wiki.apache.org/couchdb/Introduction_to_CouchDB_views>`_.
In __init__.py:

.. code-block:: python
   :linenos:

    from pyramid.events import ApplicationCreated

    @subscriber(ApplicationCreated)
    def application_created_subscriber(event):
        settings = event.app.registry.settings
        db = settings['couchdb.server'].get_or_create_db(settings['couchdb.db'])

        try:
            """Test to see if our view exists.
            """
            db.view('lists/pages')
        except ResourceNotFound:
            design_doc = {
                '_id': '_design/lists',
                'language': 'javascript',
                'views': {
                    'pages': {
                        'map': '''
                            function(doc) {
                                if (doc.doc_type === 'Page') {
                                    emit([doc.page, doc._id], null)
                                }
                            }
                        '''
                    }
                }
            }
            db.save_doc(design_doc)

CouchDB Documents
-----------------

Now we can let's add some data to a document for our home page in a CouchDB
document in our view code if it doesn't exist:

.. code-block:: python
    :linenos:

    import datetime

    from couchdbkit import *

    class Page(Document):
        author = StringProperty()
        page = StringProperty()
        content = StringProperty()
        date = DateTimeProperty()

    @view_config(route_name='home', renderer='templates/mytemplate.pt')
    def my_view(request):

        def get_data():
            return list(request.db.view('lists/pages', startkey=['home'], \
                    endkey=['home', {}], include_docs=True))

        page_data = get_data()

        if not page_data:
            Page.set_db(request.db)
            home = Page(
                author='Wendall',
                content='Using CouchDB via couchdbkit!',
                page='home',
                date=datetime.datetime.utcnow()
            )
            # save page data
            home.save()
            page_data = get_data()

        doc = page_data[0].get('doc')

        return {
            'project': 'pyramid_couchdb_example',
            'info': request.db.info(),
            'author': doc.get('author'),
            'content': doc.get('content'),
            'date': doc.get('date')
        }

Then update your home template again to add your custom values:

.. code-block:: html
   :linenos:

    <p>
        ${author}<br />
        ${content}<br />
        ${date}<br />
    </p>
