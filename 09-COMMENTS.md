### Comments

* Let's generate the comment entity inside ModelBundle:Comment.
```
php app/console generate
```
* We set a notBlank validation to authorName and tot he body.

* We create a relationship between comment and post.
```
    /**
     * @var Post
     *
     * @ORM\ManytoOne(targetEntity="Post", inversedBy="comments")
     * @ORM\JoinColumn(name="postId", referencedColumnName="id", nullable=false)
     * @Assert\NotBlank
     */
    private $post;
```

* Generates entity classes and method for the comment entity:
```
php app/console doctrine:generate:entities ModelBundle:Comment
```

* Add relationship with the comment in the post entity which file name is Post.php
```
    /**
     * @var ArrayCollection
     *
     * @ORM\OneToMany(targetEntity="Comment", mappedBy="post", cascade={"remove"})
     */
    private $comments;
```

* I will generate entities for Post 
```
php app/console doctrine:generate:entities ModelBundle:Post
```

* I will transfer this change made on entities to the database.
```
php app/console doctrine:migrations:diff
php app/console doctrine:migrations:migrate
```

* Fixture for the comments.
For the comments I will create a fixture stored in a file named 20-Comments.php
```
<?php

namespace Blog\ModelBundle\DataFixtures\ORM;

use Doctrine\Common\DataFixtures\AbstractFixture;
use Doctrine\Common\DataFixtures\OrderedFixtureInterface;
use Doctrine\Common\Persistence\ObjectManager;

/**
 * Fixtures for the Comment Entity
 */
class Comments extends AbstractFixtures implements OrderedFixturesInterface
{
	/**
	 * {@inheritDoc}
	 */
	public function getOrder()
	{
		return 20;
	}

	/**
	 * {@inheritDoc}
	 */
	public function load(ObjectManager $manager)
	{
		$posts = $manager->getRepository('ModelBundle:Post')->findAll();

		$comments = array(
			0 => '',
			1 => '',
			2 => ''
		);

		$i = 0;

		foreach($posts as $post) {
			$comment = new Comment();
			$comment->setAuthorName('someone');
			$comment->setBody($comments[$i++]);
			$comment->setPost($post);

			$manager->persist($comment);
		}

		$manager->flush();
	}
}
```
* I will load this fixture running the following command:
```
php app/console doctrine:fixture:load
```

* I will write a test to show the comments editing the file named PostControllerTest.php
```
$this->assertGreaterThanOrEqual(1, $crawler->filter('article.comment')->count(), 'There should be at least 1 comment.');

```

* I will edit Post/show.html.twig to display the comments using a partial which file name will be `_comment.html.twig`:

```

	<a id="comments"></a>
	<h2>{{ 'comment.plural' | trans }}</h2>

	{% for comment in post.comments %}
		{{ include('CoreBundle:Post:_comment.html.twig', { comment: comment }) }}
	{% endfor %}
```

The file `_comment.html.twig` will look like this:
```
<article class="comment">
	<header>
		<p>
			{{ 'on' | trans | capitalize }} <time datetime="{{ comment.createdAt | date('c') }}">{{ comment.createdAt | date }}</time>
			{{ 'by' | trans }} {{ comment.authorName }}
		</p>
	</header>

	<p>{{ comment.body | nl2br }}</p>
</article>
```

#### Display the number of comments in the posts list.
To display the number of comments in the posts list I will edit `Post/_post.html.twig`.
```
- <a href="{{ path('blog_core_post_show', {slug: post.slug}) }}#comments">
	{% set count = post.comments | length %}
	{{ 'post.comments' | transchoice(count) }}
</a>
```

#### Test the creation of a new comment

* First we will write the code to test the creation of a new comment.
I will edit `PostControllerTest.php`.
```
    /**
     * tests create a comment
     */
    public function testCreateComment()
    {
        $client = static::createClient();

        /* @var Post $post */ 
        $post = $client->getContainer()
            ->get('doctrine')
            ->getManager()
            ->getRepository('ModelBundle:Post')
            ->findFirst();

        $crawler = $client->request('GET', '/'.$post->getSlug());

        $buttonCrawlerNode = $crawler->selectButton('Send');

        $form = buttonCrawlerNode->form(array(
            'blog_modelbundle_comment[authorName]' => 'A humble commenter',
            'blog_modelbundle_comment[body]' => 'Hi, I am commenting about the following post'
        ));

        $client->submit($form);

        $this->assertTrue(
            $client->getResponse()->isRedirect('/'.$post->getSlug()),
            'There was not redirection after submitting the form'
        );

        $crawler = $client->followRedirect();

        $this->assertCount(
            1,
            $crawler->filter('html:contains("your comment was submitted successfully")').
            'There was not any confirmation message'
        );
    }
```

#### Form type
I will create a new form type class based on a Doctrine entity passing the parameter `ModelBundle:Comment to `php app/console doctrine:generate:form` command.
I will change the generated form type which file is named `CommentType.php` in this way.
```
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder
            ->add('authorName', null, array('label' => 'name'))
            ->add('body', null, array('label' => 'comment.singular'))
            ->add('post', 'submit', array('label' => 'send'));
    }
```

##### Post Controller.
* I will edit the PostController file named `PostController.php` to sent to the template the new form view.
```
        $form = $this->createForm(new CommentType());

        return array(
            'post' => $post,
            'form' => $form->createView()
        );
```
* I will edit the PostController file named `PostController.php` to create a new action named `createCommentAction`.
```
    /**
     * Create comment
     *
     * @param Request $request
     * @param string $slug
     *
     * @throws NotFoundHttpException
     * @return array
     *
     * @Route("/{slug}/create-comment")
     * @Method("POST")
     * @Template("CoreBundle:Post:show.html.twig")
     */
    public function createCommentAction(Request $request, $slug)
    {
        return array();
    }
```
* I will edit the file named `Post/show.html.twig`to display the form.
```
	<h4>{{ 'comment.write' | trans }}:</h4>
	{{ form(form, {action: path('blog_core_post_createcomment', {slug: post.slug}) ~ '#comments' }) }}
```

* I will edit the file named `main.css to improve the look and feel of the comment form.
```
form input {
	width: 300px;
}

form label {
	display: block;
	font-style: italic;
}

form textarea {
	width: 500px;
	height: 150px;
}

form ul li {
	list-style-type: none;
	color: #ff0000;
}
```

* Create the form creation.
I will edit the PostController in oder to implement the form creation.
```
    /**
     * Create comment
     *
     * @param Request $request
     * @param string $slug
     *
     * @throws NotFoundHttpException
     * @return array
     *
     * @Route("/{slug}/create-comment")
     * @Method("POST")
     * @Template("CoreBundle:Post:show.html.twig")
     */
    public function createCommentAction(Request $request, $slug)
    {
        $post = $this->getDoctrine()
            ->getRepository('ModelBundle:Post')
            ->findOneBy(
                array(
                    'slug' => $slug
                )
        );

        if (null === $post) {
            throw $this->createNotFoundHttpException('Post was not found');
        }

        $comment = new Comment();
        $comment->setPost($post);

        $form = $this->createForm(new CommentType, $comment);
        $form->handleRequest($request);

        if ($form->isValid()) {
            $this->getDoctrine()->getManager()->persist($comment);
            $this->getDoctrine()->getManager()->flush();
            $this->get('session')->getFlashBag()->add('success', 'Your comment was submit successfully');

            return $this.redirect($this->generateUrl('blog_core_post_show'), array('slug' => $post->getSlug()));
        }

        return array(
            'post' => $post,
            'form' => $form->createView()
        );
    }
```
* Setup the confirmation message.
To display the confirmation message after posting the comment I will edit the file placed into `Post/layout.html.twig`.
```
	<section>
		{% for type, message in app.session.flashbag.all() %}
			{% for message in messages %}
				<p class="session-message">{{ message }}</p>
			{% endfor %}
		{% endfor %}

		{% block section %}{% endblock %}
	</section>
```

