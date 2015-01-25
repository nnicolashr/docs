Simple aplicación de Autenticación y Autorización
#################################################

.. note::
    Esta página todavía no ha sido traducida y pertenece a la documentación de
    CakePHP 2.X. Si te animas puedes ayudarnos `traduciendo la documentación
    desde Github <https://github.com/cakephp/docs>`_.

Siguiendo nuestro ejemplo :doc:`/tutorials-and-examples/blog/blog`, imaginemos que queremos asegurar
el acceso a ciertas URLs, basado en el inicio de sesión. También tenemos otro requisito: permitir que
nuestro blog tenga múltiples autores que pueden crear, editar y eliminar sus propios artículos, pero además
no permitir que otros autores puedan realizar cambios en artículos que no les pertenecen.

Creando el código de todos los Usuarios-Relacionados
====================================================

En primer lugar, vamos a crear una nueva tabla en nuestra base de datos blog para mantener los datos de
nuestros usuarios::

    CREATE TABLE users (
        id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
        username VARCHAR(50),
        password VARCHAR(255),
        role VARCHAR(20),
        created DATETIME DEFAULT NULL,
        modified DATETIME DEFAULT NULL
    );

Nos hemos adherido a las convenciones de CakePHP en el uso de los nombres de las tablas, pero además estamos
aprovechando otra convención: Al utilizar las columnas de nombre de usuario y contraseña de la tabla de usuarios,
CakePHP podrá configurar automáticamente la mayoría de las funcionalidades asociadas en la aplicación de la
conexión del usuario.

El siguiente paso es crear nuestra tabla Usuarios, responsable de buscar, grabar y
validar los datos de usuario::

    // src/Model/Table/UsersTable.php
    namespace App\Model\Table;

    use Cake\ORM\Table;
    use Cake\Validation\Validator;

    class UsersTable extends Table
    {

        public function validationDefault(Validator $validator)
        {
            return $validator
                ->notEmpty('username', 'A username is required')
                ->notEmpty('password', 'A password is required')
                ->notEmpty('role', 'A role is required')
                ->add('role', 'inList', [
                    'rule' => ['inList', ['admin', 'author']],
                    'message' => 'Please enter a valid role'
                ]);
        }

    }

También vamos a crear nuestro UsersController. El siguiente contenido corresponde a
partes de una clase básica UsersController usando la utilidad de generación de código Bake
que viene con CakePHP::

    // src/Controller/UsersController.php

    namespace App\Controller;

    use App\Controller\AppController;
    use Cake\Error\NotFoundException;
    use Cake\Event\Event;

    class UsersController extends AppController
    {

        public function beforeFilter(Event $event)
        {
            parent::beforeFilter($event);
            $this->Auth->allow('add');
        }

         public function index()
         {
            $this->set('users', $this->Users->find('all'));
        }

        public function view($id)
        {
            if (!$id) {
                throw new NotFoundException(__('Invalid user'));
            }

            $user = $this->Users->get($id);
            $this->set(compact('user'));
        }

        public function add()
        {
            $user = $this->Users->newEntity($this->request->data);
            if ($this->request->is('post')) {
                if ($this->Users->save($user)) {
                    $this->Flash->success(__('The user has been saved.'));
                    return $this->redirect(['action' => 'add']);
                }
                $this->Flash->error(__('Unable to add the user.'));
            }
            $this->set('user', $user);
        }

    }

De la misma manera que hemos creado la vista de nuestros artículos o utilizando la generación de código,
podemos implementar las vistas de los usuarios. Para el propósito de este
tutorial, vamos a mostrar sólo la add.ctp:

.. code-block:: php

    <!-- src/Template/Users/add.ctp -->
    <div class="users form">
    <?= $this->Form->create($user) ?>
        <fieldset>
            <legend><?= __('Add User') ?></legend>
            <?= $this->Form->input('username') ?>
            <?= $this->Form->input('password') ?>
            <?= $this->Form->input('role', [
                'options' => ['admin' => 'Admin', 'author' => 'Author']
            ]) ?>
       </fieldset>
    <?= $this->Form->button(__('Submit')); ?>
    <?= $this->Form->end() ?>
    </div>

Autenticación (Login y Logout)
=================================

Ahora estamos listos para añadir a nuestra capa de autenticación. En CakePHP esto es manejado por
el :php:class:`Cake\\Controller\\Component\\AuthComponent`, una clase responsable
para exigir login a ciertas acciones, la manipulación de login y logout, y
también autoriza el inicio de sesión a los usuarios para las acciones que se les ha permitido.

Para añadir este componente a su aplicación abra su archivo ``src/Controller/AppController.php``
y añada las siguientes líneas::

    // src/Controller/AppController.php

    namespace App\Controller;

    use Cake\Event\Event;
    use Cake\Controller\Controller;

    class AppController extends Controller
    {
        //...

        public $components = [
            'Flash',
            'Auth' => [
                'loginRedirect' => [
                    'controller' => 'Articles',
                    'action' => 'index'
                ],
                'logoutRedirect' => [
                    'controller' => 'Pages',
                    'action' => 'display',
                    'home'
                ]
            ]
        ];

        public function beforeFilter(Event $event)
        {
            $this->Auth->allow(['index', 'view']);
        }
        //...
    }

No hay mucho de configurar, ya que utilizamos las convenciones de la tabla users.
Establecimos las direcciones URL que se cargarán después de realizadas las acciones de login y logout,
en nuestro caso ``/articles/`` and ``/`` respectivamente.

Lo que hicimos en la función ``beforeFilter`` fue contar qie el AuthComponent no
requiera un login para todas las acciones ``index`` and ``view``, en cada controlador. Queremos
que nuestros visitantes sean capaces de leer y listar las entradas sin registrarse en el
sitio.

Ahora, tenemos que ser capaces de registrar nuevos usuarios, guardar su nombre de usuario y contraseña,
y lo más importante, el hash de su contraseña para que no se almacene como texto sin formato en
nuestra base de datos. Digamos que el AuthComponent dejará el acceso a los usuarios no-autenticados a
los usuarios que agregan función e implementar la acción de login y logout::

    // src/Controller/UsersController.php

    public function beforeFilter(Event $event)
    {
        parent::beforeFilter($event);
        // Allow users to register and logout.
        // You should not add the "login" action to allow list. Doing so would
        // cause problems with normal functioning of AuthComponent.
        $this->Auth->allow(['add', 'logout']);
    }

    public function login()
    {
        if ($this->request->is('post')) {
            $user = $this->Auth->identify();
            if ($user) {
                $this->Auth->setUser($user);
                return $this->redirect($this->Auth->redirectUrl());
            }
            $this->Flash->error(__('Invalid username or password, try again'));
        }
    }

    public function logout()
    {
        return $this->redirect($this->Auth->logout());
    }

El Hash de contraseñas no está hecho todavía, necesitamos una clase Entity para nuestro usuario con el fin
manejar su propia lógica específica. Crea el archivo entidad ``src/Model/Entity/User.php``
y añada lo siguiente::

    // src/Model/Entity/User.php
    namespace App\Model\Entity;

    use Cake\ORM\Entity;
    use Cake\Auth\DefaultPasswordHasher;

    class User extends Entity
    {

        // Make all fields mass assignable for now.
        protected $_accessible = ['*' => true];

        // ...

        protected function _setPassword($password)
        {
            return (new DefaultPasswordHasher)->hash($password);
        }

        // ...
    }

Now every time the password property is assigned to the user it will be hashed
using the ``DefaultPasswordHasher`` class.  We're just missing a template view
file for the login function. Open up your ``src/Template/Users/login.ctp`` file
and add the following lines:

.. code-block:: php

    <!-- src/Template/Users/login.ctp -->

    <div class="users form">
    <?= $this->Flash->render('auth') ?>
    <?= $this->Form->create() ?>
        <fieldset>
            <legend><?= __('Please enter your username and password') ?></legend>
            <?= $this->Form->input('username') ?>
            <?= $this->Form->input('password') ?>
        </fieldset>
    <?= $this->Form->button(__('Login')); ?>
    <?= $this->Form->end() ?>
    </div>

You can now register a new user by accessing the ``/users/add`` URL and log in with the
newly created credentials by going to ``/users/login`` URL. Also, try to access
any other URL that was not explicitly allowed such as ``/articles/add``, you will see
that the application automatically redirects you to the login page.

And that's it! It looks too simple to be true. Let's go back a bit to explain what
happened. The ``beforeFilter`` function is telling the AuthComponent to not require a
login for the ``add`` action in addition to the ``index`` and ``view`` actions that were
already allowed in the AppController's ``beforeFilter`` function.

The ``login`` action calls the ``$this->Auth->identify()`` function in the AuthComponent,
and it works without any further config because we are following conventions as
mentioned earlier. That is, having a Users table with a username and a password
column, and use a form posted to a controller with the user data. This function
returns whether the login was successful or not, and in the case it succeeds,
then we redirect the user to the configured redirection URL that we used when
adding the AuthComponent to our application.

The logout works by just accessing the ``/users/logout`` URL and will redirect
the user to the configured logoutUrl formerly described. This URL is the result
of the ``AuthComponent::logout()`` function on success.

Authorization (who's allowed to access what)
============================================

As stated before, we are converting this blog into a multi-user authoring tool,
and in order to do this, we need to modify the articles table a bit to add the
reference to the Users table::

    ALTER TABLE articles ADD COLUMN user_id INT(11);

Also, a small change in the ArticlesController is required to store the currently
logged in user as a reference for the created article::

    // src/Controller/ArticlesController.php

    public function add()
    {
        $article = $this->Articles->newEntity($this->request->data);
        if ($this->request->is('post')) {
            // Added this line
            $article->user_id = $this->Auth->user('id');
            if ($this->Articles->save($article)) {
                $this->Flash->success(__('Your article has been saved.'));
                return $this->redirect(['action' => 'index']);
            }
            $this->Flash->error(__('Unable to add your article.'));
        }
        $this->set('article', $article);
    }

The ``user()`` function provided by the component returns any column from the
currently logged in user. We used this method to add the data into the request
info that is saved.

Let's secure our app to prevent some authors from editing or deleting the
others' articles. Basic rules for our app are that admin users can access every
URL, while normal users (the author role) can only access the permitted actions.
Again, open the AppController class and add a few more options to the Auth
config::

    // src/Controller/AppController.php

    public $components = [
        'Flash',
        'Auth' => [
            'loginRedirect' => [
                'controller' => 'Articles',
                'action' => 'index'
            ],
            'logoutRedirect' => [
                'controller' => 'Pages',
                'action' => 'display',
                'home'
            ],
            'authorize' => ['Controller'] // Added this line
        ]
    ];

    public function isAuthorized($user)
    {
        // Admin can access every action
        if (isset($user['role']) && $user['role'] === 'admin') {
            return true;
        }

        // Default deny
        return false;
    }

We just created a very simple authorization mechanism. In this case the users
with role ``admin`` will be able to access any URL in the site when logged in,
but the rest of them (i.e the role ``author``) can't do anything different from
not logged in users.

This is not exactly what we wanted, so we need to supply more rules to
our ``isAuthorized()`` method. But instead of doing it in AppController, let's
delegate each controller to supply those extra rules. The rules we're going to
add to ArticlesController should allow authors to create articles but prevent the
edition of articles if the author does not match. Open the file ``ArticlesController.php``
and add the following content::

    // src/Controller/ArticlesController.php

    public function isAuthorized($user)
    {
        // All registered users can add articles
        if ($this->request->action === 'add') {
            return true;
        }

        // The owner of an article can edit and delete it
        if (in_array($this->request->action, ['edit', 'delete'])) {
            $articleId = (int)$this->request->params['pass'][0];
            if ($this->Articles->isOwnedBy($articleId, $user['id'])) {
                return true;
            }
        }

        return parent::isAuthorized($user);
    }

We're now overriding the AppController's ``isAuthorized()`` call and internally
checking if the parent class is already authorizing the user. If he isn't,
then just allow him to access the add action, and conditionally access
edit and delete. One final thing has not been implemented. To tell whether
or not the user is authorized to edit the article, we're calling a ``isOwnedBy()``
function in the Articles table. Let's then implement that function::

    // src/Model/Table/ArticlesTable.php

    public function isOwnedBy($articleId, $userId)
    {
        return $this->exists(['id' => $articleId, 'user_id' => $userId]);
    }

This concludes our simple authentication and authorization tutorial. For securing
the UsersController you can follow the same technique we did for ArticlesController.
You could also be more creative and code something more general in AppController based
on your own rules.

Should you need more control, we suggest you read the complete Auth guide in the
:doc:`/core-libraries/components/authentication` section where you will find more
about configuring the component, creating custom Authorization classes, and much more.

Suggested Follow-up Reading
---------------------------

#. :doc:`/console-and-shells/code-generation-with-bake` Generating basic CRUD code
#. :doc:`/core-libraries/components/authentication`: User registration and login

.. meta::
    :title lang=en: Simple Authentication and Authorization Application
    :keywords lang=en: auto increment,authorization application,model user,array,conventions,authentication,urls,cakephp,delete,doc,columns
