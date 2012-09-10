
Testing Django may seems complicated sometimes, and especially with some django features full of
encapsulation. This can be easily fixed by some python tricks.

Testing an admin function
-------------------------

Admin classes are initialized by Django directly. The trick here is to override the init method.

    def setUp(self):
        MyAdminClass.__init__ = lambda x: None
        my_admin_object = MyAdminClass()


Testing a template tag
----------------------

The same kind of trick can be applied to test template tag with parameters.

    def new_init(self, value, user):
        self.value = value
        self.user = user

    def setUp(self):
        MyTemplateTagNode.__init__ = new_init
        value = "my value"
        my_user = User.objects.get(id=1)
        node = MyTemplateTagNode(my_value, my_user))

Testing complex view
--------------------

Sometimes some view may rely on external services or request and you don't want to rely
on anything external when performing unit tests.

    class MyComplexView(BaseView):
    
         det get(self, *args, **kwargs):
              ...
              my_external_object = ExternalObject()
              ...

You will need sometimes to redesign your code especially to be able to test it.

    class MyComplexView(BaseView):

        def get_external_object(self):
            return ExternalObject()
    
         det get(self, *args, **kwargs):
              ...
              my_external_object = self.get_external_object()
              ...

Then, in your setup method, just do:

    def get_external_object(self):
        # Return an object implementing the right protocol
        return MockExternalObject()

    def setup(self):
        MyComplexView.get_external_object = get_external_object


Testing dynamic type creation
-----------------------------

If you need to test an exception is raised when a class in defined (Imagine something like django model)
you will have to use the type trick.

Encapsulate your class creation in a function (sample extract from the csv_importer application):

    def create_unexpected_model():
        return type('TestCsvDBUnmatchingModel', CsvModel, {'name': CharField(unexpected=True)})

then you can use assertRaises

    self.assertRaises(ValueError, create_unexpected_model)    


Handling data
-----------------

On a growing project, managing data through fixture can be really painful.
Easy with django-dynamic-fixture.

A good advice is to create your own service to wrap the data creation into a more readable and
usable API usage.

    user = GenerateUser().create_admin()

Testing email content and formatting
------------------------------------

By default, django override some settings when using emails in testing. If you need to automatically generate email through a smtp server, you will have to change the settings just before performing the test.

For this, I use the following context manager:

    class EmailSettingChange(object):
	    """ 
	    Change the email setting to not use the locmem backend
	    but send real email
	    """

	    def __init__(self):
		self.original_email_backend = settings.EMAIL_BACKEND

	    def __enter__(self):
		if hasattr(settings, 'TEST_SEND_EMAIL') and settings.TEST_SEND_EMAIL:
		    settings.EMAIL_BACKEND = settings.TEST_SEND_EMAIL_CONF['EMAIL_BACKEND']
		    settings.EMAIL_HOST = settings.TEST_SEND_EMAIL_CONF['EMAIL_HOST']
		    settings.EMAIL_HOST_USER = settings.TEST_SEND_EMAIL_CONF['EMAIL_HOST_USER']
		    settings.EMAIL_HOST_PASSWORD = settings.TEST_SEND_EMAIL_CONF['EMAIL_HOST_PASSWORD']
		    settings.EMAIL_PORT = settings.TEST_SEND_EMAIL_CONF['EMAIL_PORT']
		    settings.EMAIL_USE_TLS = settings.TEST_SEND_EMAIL_CONF['EMAIL_USE_TLS']

	    def __exit__(self, type, value, tb):
		settings.EMAIL_BACKEND = self.original_email_backend

Testing rollback
----------------

If you need to test than rollbacking is working correctly in your application, you can't do that with a django test case, you have to use a TransactionTestCase. Take care to manually commit at the end of your setup method.


More information in the [django doc](https://docs.djangoproject.com/en/dev/topics/testing/#django.test.TransactionTestCase)

