.. index::
   single: Cache (ön bellek)

HTTP Ön Belleği(cache)
======================

Zengin web uygulamalarının doğası dinamik olmalarıdır. Uygulamanız ne kadar
verimli olursa olsun her istek daima bir statik dosyanın servis edilmesinden
daha çok maliyetlidir.

Ancak çoğu web uygulamalarında bu bir sorun değildir. Symfony2 şimşek 
hızındadır ve siz çok ağır işler yaptırmadıkça her istek sunucunuza fazla
yüklenmeden anında geri döndürülür.

Fakat siteniz büyüdüğünde bu fazlalıklar problem olmaya başlar. İşlemler,
her istekte normal olarak sadece bir kez olmalıdır. İşte tam da caching
(önbellekleme)  işleminin işi budur.

Devlerin Omuzlarında Ön Bellekleme
----------------------------------

Uygulamanın performansını arttırmanın en etkili yolu, tüm sayfanın çıktısını
önbelleğe(cache) almak ve sonraki her istekte bu ön bellek çıktısını vermektir.
Elbette bu her zaman yüksek oranda içerikleri değişen web sitelerinde
sürekli mümkün olmamaktadır. Bu bölümde Symfony2 cache sisteminin nasıl
çalıştığını gösterecek ve neden bunun mümkün olan  en iyi yaklaşım olarak 
düşünüdüğümüzü göreceksiniz.

Symfony2 cache sistemi :term:`HTTP specification` sistemi olarak tanımlanan
basit HTTP cache sisteminden gücünü aldığı için farklıdır. Cache metodolojisini
yeniden keşfetmektense Symfony2 standart olarak Web üzerindeki basit iletişimi
benimser. HTTP doğrulama(validation) ve sonlu zamanlı(expiration) cache modellerinin
temellerini anladığınızda, Symfony2 cache sisteminin uzmanı olmaya hazır olacaksınız.

Symfony2 ile nasıl önbellekleme (cache) yapılacağını öğrenmek için,
bu konuyu 4 adımda ele alacağız:

* **Adım 1**: :ref:`gateway cache <gateway-caches>`, ya da reverse proxy 
  uygulamanızın ön kısmına oturan bağımsız bir katmandır.  Reverse proxy
  response'ları (cevapları) uygulamanızdan döndüğü gibi önbellekler ve 
  istekler için bu istekler uygulamanıza ulaşmadan önce önbeleklenmiş 
  cevapları(response) döndürür. Symfony2 kendi reverse proxy'sini sağlar
  ancak herhangi diğer bir reverse proxy sistemide kullanılabilir.

* **Adım 2**: Uygulamanız ve istemci arasındaki gateway cache ve diğer
  cache'lerin iletişimi için :ref:`HTTP cache <http-cache-introduction>`
  başlıkları kullanılır.

* **Adım 3**: Cache içerisine alınmış bir içeriğin *taze* (cache tarafından
  yeniden kullanılabilir) ya da *bayat*(uygulama tarafından yeniden yaratılması
  gereken içeik) olup olmadığını anlamak için HTTP :ref:`expiration(zaman kısıtlı) 
  ve validation(doğrulama)<http-expiration-validation>` metodları kullanılır.

* **Adım 4**: :ref:`Edge Side Includes <edge-side-includes>` (ESI) HTTP cache'ını
  bağımsız olarak önbellek parçaları haliden kullanılmasına(hatta iç içe parçalara)
  olanak sağlar. ESI ile sayfanın tamamını 60 dakikada bir önbellekleyebilirsiniz
  fakat gömülü kenar kutuları(sidebar) sadece 5 dakika da bir önbelleklenebilir.
  
HTTP ile önbellekleme Symfony2 ile ilk kez gelen bir özellik olmamasına
rağmen bu konuda pek çok makale bulunmaktadır. Eğer HTTP önbelleklemesi
konusunda yeni iseniz Ryan Tomayko'nun `Things Caches Do`_ adlı makalesini
okumanızı şiddetle tavsiye ederiz. Bu konudaki başka bir güzel makale ise 
Mark Nottingham'ın `Cache Tutorial`_ 'ıdır.

.. index::
   single: Cache; Proxy
   single: Cache; Reverse Proxy
   single: Cache; Gateway

.. _gateway-caches:

Gateway Cache ile Önbelleklemek
-------------------------------

HTTP ile cache yaparken *cache* uygulamanızdan tamamen ayrıdır ve uygulamanız
ile istek yapan istemci arasında durur. 

Cache'in işi istemciden gelen istekleri kabul etmek ve onları uygulamaya
geri göndermektir. Cache ayrıca uygulamanızdan istekleri geri alacak ve
onları istemciye iletecektir. Cache istemci ile uygulamanız arasındaki
istek-cevap iletişimindeki "aracı" dır.

Bu arada cache "cache edilebilir" gördüğü her istedği saklayacaktır
(bkz :ref:`http-cache-introduction`). Eğer aynı istek yeniden yapılıysa
cache uygulamanızı yok sayarak cache edilmiş isteği istemciye yollayacaktır.

Bu tip cache HTTP gateway cache olarak adlandırılır ve `Varnish`_, 
`Reverse proxy mode da Squid`_, ve Symfony2 reverse proxy gibi pek çok 
şekli mevcuttur.

.. index::
   single: Cache; Tipler

Cache Tipleri
~~~~~~~~~~~~~

Ancak bir gateway cache tek cache tipi değildir. Aslında uygulamanız
tarafından gönderilen HTTP cache başlıkları üç tip farklı cache tarafından
yorumlanır ve işlenir:

* *Browser cache 'leri*: Her browser "geri" hareketi ya da resimler ve diğer
  varlıklar için kendisinin kullandığı bir yerel cache sistemi ile birlikte
  gelir. Browser cache 'i, cache bilgisini başkalarınla paylaşmayan *özel* 
  (private)  bir cache 'dir.
  
* *Proxy cache'leri*: Bir proxy cache pek çok insan tarafından *paylaşılan*
  tek bir kaynaktır. Bu genellikle büyük kurumlar ve ISS'ler tarafından
  gecikme (latency) ve ağ trafiğini azaltmak için kullanılır.

* *Gateway cache'leri*: Proxy gibi buda *paylaşılan* bir cache'dir. Ancak bu
  sunucu tarafında gerçekleşir. Ağ yöneticileri tarafından web sitelerini
  daha ölçeklenebilir, güvenilir ve performanslı yapmak amacıyla kurulur.
 
.. tip::

    Gateway cache'leri bazen reverse proxy cache, surrogate cache hatta
    HTTP hızlandırıcıları olarak da bilinir.

.. note::

    *Private* ile *shared* cache arasındaki mana farkı özellikle bir 
    kullanıcıya ait veriyi barındıran içeriklerin (örn: hesap bilgisi)
    cache'lenmesi şeklinde işlediğimizde daha da netlik kazanacaktır.

Uygulamanızdaki her response muhtemelen bir ya da ilk iki cache tipi 
şeklinde gidecektir. Bu cache'ler kontrolunuz dışındadır ancak response
içerisine konulmuş HTTP cache direktiflerini izleyebilirsiniz.

.. index::
   single: Cache; Symfony2 Reverse Proxy

.. _`symfony-gateway-cache`:

Symfony2 Reverse Proxy
~~~~~~~~~~~~~~~~~~~~~~

Symfony2 PHP'de yazılmış bir reverse proxy (gatewa cache olarak da anılır)
ile birlikte gelir. Bunu devreye alarak uygulamanızdan cache'lenebilir 
cevapları alarak cache işine başlayabilirsiniz. Kurulumu oldukça kolaydır.
Her Symfony2 uygulaması önceden konfigüre edilmiş (``AppKernel``) 
'i kullanan bir cache çekirdeği (``AppCache``) ile birlikte gelir. Cache
çekirdeği *reverse proxy* 'dir.

Cache 'lemeyi aktif edebilmek için fontro controller'ınızı cache kernelini
aktif edecek şekilde aşağıdaki gibi düzenleyin::

    // web/app.php

    require_once __DIR__.'/../app/bootstrap.php.cache';
    require_once __DIR__.'/../app/AppKernel.php';
    require_once __DIR__.'/../app/AppCache.php';

    use Symfony\Component\HttpFoundation\Request;

    $kernel = new AppKernel('prod', false);
    $kernel->loadClassCache();
    // wrap the default AppKernel with the AppCache one
    $kernel = new AppCache($kernel);
    $kernel->handle(Request::createFromGlobals())->send();

Cache kerneli anında reverse proxy olarak uygulamanızdan cevapları cache'leyip
istemciye göndermeye başlayacaktır.

.. tip::

    Cache çekirdeği cache katmanında neler olduğunu bildiren ``getLog()``
    adında özel bir metoda sahiptir. Geliştirme ortamında (environment)
    bunu cache stratejinizin doğru çalışıp çalışmadığını kontrol etmek
    için kullanabilirsiniz::

        error_log($kernel->getLog());

``AppCache`` nesnesi tutarlı bir konfigürasyona sahiptir ancak bazı 
ayarlara ince ayar çekebilmek için ``getOptions()`` metodu ile bu ayarları
düzenleyebilirsiniz::

    // app/AppCache.php

    use Symfony\Bundle\FrameworkBundle\HttpCache\HttpCache;

    class AppCache extends HttpCache
    {
        protected function getOptions()
        {
            return array(
                'debug'                  => false,
                'default_ttl'            => 0,
                'private_headers'        => array('Authorization', 'Cookie'),
                'allow_reload'           => false,
                'allow_revalidate'       => false,
                'stale_while_revalidate' => 2,
                'stale_if_error'         => 60,
            );
        }
    }

.. tip::

    ``getOption()`` metodu ile seçenek değiştirilmedikçe ``debug`` seçeneği
    otomatik olarak ``AppKernel`` 'deki değeri kendisine atayacaktır.

Aşağıda ana seçeneklerin listesi verilmiştir:

* ``default_ttl``: Cache girdisinin response tarafından belirli bir 
  taze kalma değeri verilmediğinde tazelik (freshness) değerini 
  saniye olarak belirler. ``Cache-Control`` ya da ``Expires`` başlıkları 
  bu değeri değiştirebilir (override) (varsayılan: ``0``);

* ``private_headers``: response'larda "özel" ``Cache-Control`` davranısının 
  tetiklediği response'un ``Cache-Control`` direktifi üzerinden 
  ``public`` ya da ``private`` durumunun belirgin olmadığı durumlarda request'in
  alacağı değerdir. (varsayılan : ``Authorization`` ve ``Cookie``);

* ``allow_reload``: İstemcinin istekte ``Cache-Control`` 'ün "no-cache" 
  direktifi ile bir cache 'i yeniden yüklenmeye zorlamasına izin verir.
  Bunun değeri 'ni RFC 2616  ile uyumlu olmasını istiyorsanız ``true``  
  yapın.(varsayılan: ``false``);

* ``allow_revalidate``: istemcinin cache'i istekte ``Cache-Control``'de 
  "max-age=0" değeri ile yeniden doğrulama yapıp yapmamasını belirler.
  Bunun değeri 'ni RFC 2616  ile uyumlu olmasını istiyorsanız ``true``  
  yapın.(varsayılan: ``false``);


* ``stale_while_revalidate``: Specifies the default number of seconds (the
  granularity is the second as the Response TTL precision is a second) during
  which the cache can immediately return a stale response while it revalidates
  it in the background (default: ``2``); this setting is overridden by the
  ``stale-while-revalidate`` HTTP ``Cache-Control`` extension (see RFC 5861);

* ``stale_if_error``: Specifies the default number of seconds (the granularity
  is the second) during which the cache can serve a stale response when an
  error is encountered (default: ``60``). This setting is overridden by the
  ``stale-if-error`` HTTP ``Cache-Control`` extension (see RFC 5861).

If ``debug`` is ``true``, Symfony2 automatically adds a ``X-Symfony-Cache``
header to the response containing useful information about cache hits and
misses.

.. sidebar:: Changing from one Reverse Proxy to Another

    The Symfony2 reverse proxy is a great tool to use when developing your
    website or when you deploy your website to a shared host where you cannot
    install anything beyond PHP code. But being written in PHP, it cannot
    be as fast as a proxy written in C. That's why we highly recommend you
    to use Varnish or Squid on your production servers if possible. The good
    news is that the switch from one proxy server to another is easy and
    transparent as no code modification is needed in your application. Start
    easy with the Symfony2 reverse proxy and upgrade later to Varnish when
    your traffic increases.

    For more information on using Varnish with Symfony2, see the
    :doc:`How to use Varnish </cookbook/cache/varnish>` cookbook chapter.

.. note::

    The performance of the Symfony2 reverse proxy is independent of the
    complexity of the application. That's because the application kernel is
    only booted when the request needs to be forwarded to it.

.. index::
   single: Cache; HTTP

.. _http-cache-introduction:

Introduction to HTTP Caching
----------------------------

To take advantage of the available cache layers, your application must be
able to communicate which responses are cacheable and the rules that govern
when/how that cache should become stale. This is done by setting HTTP cache
headers on the response.

.. tip::

    Keep in mind that "HTTP" is nothing more than the language (a simple text
    language) that web clients (e.g. browsers) and web servers use to communicate
    with each other. When we talk about HTTP caching, we're talking about the
    part of that language that allows clients and servers to exchange information
    related to caching.

HTTP specifies four response cache headers that we're concerned with:

* ``Cache-Control``
* ``Expires``
* ``ETag``
* ``Last-Modified``

The most important and versatile header is the ``Cache-Control`` header,
which is actually a collection of various cache information.

.. note::

    Each of the headers will be explained in full detail in the
    :ref:`http-expiration-validation` section.

.. index::
   single: Cache; Cache-Control header
   single: HTTP headers; Cache-Control

The Cache-Control Header
~~~~~~~~~~~~~~~~~~~~~~~~

The ``Cache-Control`` header is unique in that it contains not one, but various
pieces of information about the cacheability of a response. Each piece of
information is separated by a comma:

     Cache-Control: private, max-age=0, must-revalidate

     Cache-Control: max-age=3600, must-revalidate

Symfony provides an abstraction around the ``Cache-Control`` header to make
its creation more manageable:

.. code-block:: php

    $response = new Response();

    // mark the response as either public or private
    $response->setPublic();
    $response->setPrivate();

    // set the private or shared max age
    $response->setMaxAge(600);
    $response->setSharedMaxAge(600);

    // set a custom Cache-Control directive
    $response->headers->addCacheControlDirective('must-revalidate', true);

Public vs Private Responses
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Both gateway and proxy caches are considered "shared" caches as the cached
content is shared by more than one user. If a user-specific response were
ever mistakenly stored by a shared cache, it might be returned later to any
number of different users. Imagine if your account information were cached
and then returned to every subsequent user who asked for their account page!

To handle this situation, every response may be set to be public or private:

* *public*: Indicates that the response may be cached by both private and
  shared caches;

* *private*: Indicates that all or part of the response message is intended
  for a single user and must not be cached by a shared cache.

Symfony conservatively defaults each response to be private. To take advantage
of shared caches (like the Symfony2 reverse proxy), the response will need
to be explicitly set as public.

.. index::
   single: Cache; Safe methods

Safe Methods
~~~~~~~~~~~~

HTTP caching only works for "safe" HTTP methods (like GET and HEAD). Being
safe means that you never change the application's state on the server when
serving the request (you can of course log information, cache data, etc).
This has two very reasonable consequences:

* You should *never* change the state of your application when responding
  to a GET or HEAD request. Even if you don't use a gateway cache, the presence
  of proxy caches mean that any GET or HEAD request may or may not actually
  hit your server.

* Don't expect PUT, POST or DELETE methods to cache. These methods are meant
  to be used when mutating the state of your application (e.g. deleting a
  blog post). Caching them would prevent certain requests from hitting and
  mutating your application.

Caching Rules and Defaults
~~~~~~~~~~~~~~~~~~~~~~~~~~

HTTP 1.1 allows caching anything by default unless there is an explicit
``Cache-Control`` header. In practice, most caches do nothing when requests
have a cookie, an authorization header, use a non-safe method (i.e. PUT, POST,
DELETE), or when responses have a redirect status code.

Symfony2 automatically sets a sensible and conservative ``Cache-Control``
header when none is set by the developer by following these rules:

* If no cache header is defined (``Cache-Control``, ``Expires``, ``ETag``
  or ``Last-Modified``), ``Cache-Control`` is set to ``no-cache``, meaning
  that the response will not be cached;

* If ``Cache-Control`` is empty (but one of the other cache headers is present),
  its value is set to ``private, must-revalidate``;

* But if at least one ``Cache-Control`` directive is set, and no 'public' or
  ``private`` directives have been explicitly added, Symfony2 adds the
  ``private`` directive automatically (except when ``s-maxage`` is set).

.. _http-expiration-validation:

HTTP Expiration and Validation
------------------------------

The HTTP specification defines two caching models:

* With the `expiration model`_, you simply specify how long a response should
  be considered "fresh" by including a ``Cache-Control`` and/or an ``Expires``
  header. Caches that understand expiration will not make the same request
  until the cached version reaches its expiration time and becomes "stale".

* When pages are really dynamic (i.e. their representation changes often),
  the `validation model`_ is often necessary. With this model, the
  cache stores the response, but asks the server on each request whether
  or not the cached response is still valid. The application uses a unique
  response identifier (the ``Etag`` header) and/or a timestamp (the ``Last-Modified``
  header) to check if the page has changed since being cached.

The goal of both models is to never generate the same response twice by relying
on a cache to store and return "fresh" responses.

.. sidebar:: Reading the HTTP Specification

    The HTTP specification defines a simple but powerful language in which
    clients and servers can communicate. As a web developer, the request-response
    model of the specification dominates our work. Unfortunately, the actual
    specification document - `RFC 2616`_ - can be difficult to read.

    There is an on-going effort (`HTTP Bis`_) to rewrite the RFC 2616. It does
    not describe a new version of HTTP, but mostly clarifies the original HTTP
    specification. The organization is also improved as the specification
    is split into seven parts; everything related to HTTP caching can be
    found in two dedicated parts (`P4 - Conditional Requests`_ and `P6 -
    Caching: Browser and intermediary caches`_).

    As a web developer, we strongly urge you to read the specification. Its
    clarity and power - even more than ten years after its creation - is
    invaluable. Don't be put-off by the appearance of the spec - its contents
    are much more beautiful than its cover.

.. index::
   single: Cache; HTTP Expiration

Expiration
~~~~~~~~~~

The expiration model is the more efficient and straightforward of the two
caching models and should be used whenever possible. When a response is cached
with an expiration, the cache will store the response and return it directly
without hitting the application until it expires.

The expiration model can be accomplished using one of two, nearly identical,
HTTP headers: ``Expires`` or ``Cache-Control``.

.. index::-
   single: Cache; Expires header
   single: HTTP headers; Expires

Expiration with the ``Expires`` Header
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

According to the HTTP specification, "the ``Expires`` header field gives
the date/time after which the response is considered stale." The ``Expires``
header can be set with the ``setExpires()`` ``Response`` method. It takes a
``DateTime`` instance as an argument::

    $date = new DateTime();
    $date->modify('+600 seconds');

    $response->setExpires($date);

The resulting HTTP header will look like this::

    Expires: Thu, 01 Mar 2011 16:00:00 GMT

.. note::

    The ``setExpires()`` method automatically converts the date to the GMT
    timezone as required by the specification.

Note that in HTTP versions before 1.1 the origin server wasn't required to
send the ``Date`` header. Consequently the cache (e.g. the browser) might
need to rely onto his local clock to evaluate the ``Expires`` header making
the lifetime calculation vulnerable to clock skew. Another limitation
of the ``Expires`` header is that the specification states that "HTTP/1.1
servers should not send ``Expires`` dates more than one year in the future."

.. index::
   single: Cache; Cache-Control header
   single: HTTP headers; Cache-Control

Expiration with the ``Cache-Control`` Header
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Because of the ``Expires`` header limitations, most of the time, you should
use the ``Cache-Control`` header instead. Recall that the ``Cache-Control``
header is used to specify many different cache directives. For expiration,
there are two directives, ``max-age`` and ``s-maxage``. The first one is
used by all caches, whereas the second one is only taken into account by
shared caches::

    // Sets the number of seconds after which the response
    // should no longer be considered fresh
    $response->setMaxAge(600);

    // Same as above but only for shared caches
    $response->setSharedMaxAge(600);

The ``Cache-Control`` header would take on the following format (it may have
additional directives)::

    Cache-Control: max-age=600, s-maxage=600

.. index::
   single: Cache; Validation

Validation
~~~~~~~~~~

When a resource needs to be updated as soon as a change is made to the underlying
data, the expiration model falls short. With the expiration model, the application
won't be asked to return the updated response until the cache finally becomes
stale.

The validation model addresses this issue. Under this model, the cache continues
to store responses. The difference is that, for each request, the cache asks
the application whether or not the cached response is still valid. If the
cache *is* still valid, your application should return a 304 status code
and no content. This tells the cache that it's ok to return the cached response.

Under this model, you mainly save bandwidth as the representation is not
sent twice to the same client (a 304 response is sent instead). But if you
design your application carefully, you might be able to get the bare minimum
data needed to send a 304 response and save CPU also (see below for an implementation
example).

.. tip::

    The 304 status code means "Not Modified". It's important because with
    this status code do *not* contain the actual content being requested.
    Instead, the response is simply a light-weight set of directions that
    tell cache that it should use its stored version.

Like with expiration, there are two different HTTP headers that can be used
to implement the validation model: ``ETag`` and ``Last-Modified``.

.. index::
   single: Cache; Etag header
   single: HTTP headers; Etag

Validation with the ``ETag`` Header
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``ETag`` header is a string header (called the "entity-tag") that uniquely
identifies one representation of the target resource. It's entirely generated
and set by your application so that you can tell, for example, if the ``/about``
resource that's stored by the cache is up-to-date with what your application
would return. An ``ETag`` is like a fingerprint and is used to quickly compare
if two different versions of a resource are equivalent. Like fingerprints,
each ``ETag`` must be unique across all representations of the same resource.

Let's walk through a simple implementation that generates the ETag as the
md5 of the content::

    public function indexAction()
    {
        $response = $this->render('MyBundle:Main:index.html.twig');
        $response->setETag(md5($response->getContent()));
        $response->isNotModified($this->getRequest());

        return $response;
    }

The ``Response::isNotModified()`` method compares the ``ETag`` sent with
the ``Request`` with the one set on the ``Response``. If the two match, the
method automatically sets the ``Response`` status code to 304.

This algorithm is simple enough and very generic, but you need to create the
whole ``Response`` before being able to compute the ETag, which is sub-optimal.
In other words, it saves on bandwidth, but not CPU cycles.

In the :ref:`optimizing-cache-validation` section, we'll show how validation
can be used more intelligently to determine the validity of a cache without
doing so much work.

.. tip::

    Symfony2 also supports weak ETags by passing ``true`` as the second
    argument to the
    :method:`Symfony\\Component\\HttpFoundation\\Response::setETag` method.

.. index::
   single: Cache; Last-Modified header
   single: HTTP headers; Last-Modified

Validation with the ``Last-Modified`` Header
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``Last-Modified`` header is the second form of validation. According
to the HTTP specification, "The ``Last-Modified`` header field indicates
the date and time at which the origin server believes the representation
was last modified." In other words, the application decides whether or not
the cached content has been updated based on whether or not it's been updated
since the response was cached.

For instance, you can use the latest update date for all the objects needed to
compute the resource representation as the value for the ``Last-Modified``
header value::

    public function showAction($articleSlug)
    {
        // ...

        $articleDate = new \DateTime($article->getUpdatedAt());
        $authorDate = new \DateTime($author->getUpdatedAt());

        $date = $authorDate > $articleDate ? $authorDate : $articleDate;

        $response->setLastModified($date);
        $response->isNotModified($this->getRequest());

        return $response;
    }

The ``Response::isNotModified()`` method compares the ``If-Modified-Since``
header sent by the request with the ``Last-Modified`` header set on the
response. If they are equivalent, the ``Response`` will be set to a 304 status
code.

.. note::

    The ``If-Modified-Since`` request header equals the ``Last-Modified``
    header of the last response sent to the client for the particular resource.
    This is how the client and server communicate with each other and decide
    whether or not the resource has been updated since it was cached.

.. index::
   single: Cache; Conditional get
   single: HTTP; 304

.. _optimizing-cache-validation:

Optimizing your Code with Validation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The main goal of any caching strategy is to lighten the load on the application.
Put another way, the less you do in your application to return a 304 response,
the better. The ``Response::isNotModified()`` method does exactly that by
exposing a simple and efficient pattern::

    public function showAction($articleSlug)
    {
        // Get the minimum information to compute
        // the ETag or the Last-Modified value
        // (based on the Request, data is retrieved from
        // a database or a key-value store for instance)
        $article = // ...

        // create a Response with a ETag and/or a Last-Modified header
        $response = new Response();
        $response->setETag($article->computeETag());
        $response->setLastModified($article->getPublishedAt());

        // Check that the Response is not modified for the given Request
        if ($response->isNotModified($this->getRequest())) {
            // return the 304 Response immediately
            return $response;
        } else {
            // do more work here - like retrieving more data
            $comments = // ...

            // or render a template with the $response you've already started
            return $this->render(
                'MyBundle:MyController:article.html.twig',
                array('article' => $article, 'comments' => $comments),
                $response
            );
        }
    }

When the ``Response`` is not modified, the ``isNotModified()`` automatically sets
the response status code to ``304``, removes the content, and removes some
headers that must not be present for ``304`` responses (see
:method:`Symfony\\Component\\HttpFoundation\\Response::setNotModified`).

.. index::
   single: Cache; Vary
   single: HTTP headers; Vary

Varying the Response
~~~~~~~~~~~~~~~~~~~~

So far, we've assumed that each URI has exactly one representation of the
target resource. By default, HTTP caching is done by using the URI of the
resource as the cache key. If two people request the same URI of a cacheable
resource, the second person will receive the cached version.

Sometimes this isn't enough and different versions of the same URI need to
be cached based on one or more request header values. For instance, if you
compress pages when the client supports it, any given URI has two representations:
one when the client supports compression, and one when it does not. This
determination is done by the value of the ``Accept-Encoding`` request header.

In this case, we need the cache to store both a compressed and uncompressed
version of the response for the particular URI and return them based on the
request's ``Accept-Encoding`` value. This is done by using the ``Vary`` response
header, which is a comma-separated list of different headers whose values
trigger a different representation of the requested resource::

    Vary: Accept-Encoding, User-Agent

.. tip::

    This particular ``Vary`` header would cache different versions of each
    resource based on the URI and the value of the ``Accept-Encoding`` and
    ``User-Agent`` request header.

The ``Response`` object offers a clean interface for managing the ``Vary``
header::

    // set one vary header
    $response->setVary('Accept-Encoding');

    // set multiple vary headers
    $response->setVary(array('Accept-Encoding', 'User-Agent'));

The ``setVary()`` method takes a header name or an array of header names for
which the response varies.

Expiration and Validation
~~~~~~~~~~~~~~~~~~~~~~~~~

You can of course use both validation and expiration within the same ``Response``.
As expiration wins over validation, you can easily benefit from the best of
both worlds. In other words, by using both expiration and validation, you
can instruct the cache to serve the cached content, while checking back
at some interval (the expiration) to verify that the content is still valid.

.. index::
    pair: Cache; Configuration

More Response Methods
~~~~~~~~~~~~~~~~~~~~~

The Response class provides many more methods related to the cache. Here are
the most useful ones::

    // Marks the Response stale
    $response->expire();

    // Force the response to return a proper 304 response with no content
    $response->setNotModified();

Additionally, most cache-related HTTP headers can be set via the single
``setCache()`` method::

    // Set cache settings in one call
    $response->setCache(array(
        'etag'          => $etag,
        'last_modified' => $date,
        'max_age'       => 10,
        's_maxage'      => 10,
        'public'        => true,
        // 'private'    => true,
    ));

.. index::
  single: Cache; ESI
  single: ESI

.. _edge-side-includes:

Using Edge Side Includes
------------------------

Gateway caches are a great way to make your website perform better. But they
have one limitation: they can only cache whole pages. If you can't cache
whole pages or if parts of a page has "more" dynamic parts, you are out of
luck. Fortunately, Symfony2 provides a solution for these cases, based on a
technology called `ESI`_, or Edge Side Includes. Akamaï wrote this specification
almost 10 years ago, and it allows specific parts of a page to have a different
caching strategy than the main page.

The ESI specification describes tags you can embed in your pages to communicate
with the gateway cache. Only one tag is implemented in Symfony2, ``include``,
as this is the only useful one outside of Akamaï context:

.. code-block:: html

    <html>
        <body>
            Some content

            <!-- Embed the content of another page here -->
            <esi:include src="http://..." />

            More content
        </body>
    </html>

.. note::

    Notice from the example that each ESI tag has a fully-qualified URL.
    An ESI tag represents a page fragment that can be fetched via the given
    URL.

When a request is handled, the gateway cache fetches the entire page from
its cache or requests it from the backend application. If the response contains
one or more ESI tags, these are processed in the same way. In other words,
the gateway cache either retrieves the included page fragment from its cache
or requests the page fragment from the backend application again. When all
the ESI tags have been resolved, the gateway cache merges each into the main
page and sends the final content to the client.

All of this happens transparently at the gateway cache level (i.e. outside
of your application). As you'll see, if you choose to take advantage of ESI
tags, Symfony2 makes the process of including them almost effortless.

Using ESI in Symfony2
~~~~~~~~~~~~~~~~~~~~~

First, to use ESI, be sure to enable it in your application configuration:

.. configuration-block::

    .. code-block:: yaml

        # app/config/config.yml
        framework:
            # ...
            esi: { enabled: true }

    .. code-block:: xml

        <!-- app/config/config.xml -->
        <framework:config ...>
            <!-- ... -->
            <framework:esi enabled="true" />
        </framework:config>

    .. code-block:: php

        // app/config/config.php
        $container->loadFromExtension('framework', array(
            // ...
            'esi'    => array('enabled' => true),
        ));

Now, suppose we have a page that is relatively static, except for a news
ticker at the bottom of the content. With ESI, we can cache the news ticker
independent of the rest of the page.

.. code-block:: php

    public function indexAction()
    {
        $response = $this->render('MyBundle:MyController:index.html.twig');
        $response->setSharedMaxAge(600);

        return $response;
    }

In this example, we've given the full-page cache a lifetime of ten minutes.
Next, let's include the news ticker in the template by embedding an action.
This is done via the ``render`` helper (See :ref:`templating-embedding-controller`
for more details).

As the embedded content comes from another page (or controller for that
matter), Symfony2 uses the standard ``render`` helper to configure ESI tags:

.. configuration-block::

    .. code-block:: jinja

        {% render '...:news' with {}, {'standalone': true} %}

    .. code-block:: php

        <?php echo $view['actions']->render('...:news', array(), array('standalone' => true)) ?>

By setting ``standalone`` to ``true``, you tell Symfony2 that the action
should be rendered as an ESI tag. You might be wondering why you would want to
use a helper instead of just writing the ESI tag yourself. That's because
using a helper makes your application work even if there is no gateway cache
installed. Let's see how it works.

When standalone is ``false`` (the default), Symfony2 merges the included page
content within the main one before sending the response to the client. But
when standalone is ``true``, *and* if Symfony2 detects that it's talking
to a gateway cache that supports ESI, it generates an ESI include tag. But
if there is no gateway cache or if it does not support ESI, Symfony2 will
just merge the included page content within the main one as it would have
done were standalone set to ``false``.

.. note::

    Symfony2 detects if a gateway cache supports ESI via another Akamaï
    specification that is supported out of the box by the Symfony2 reverse
    proxy.

The embedded action can now specify its own caching rules, entirely independent
of the master page.

.. code-block:: php

    public function newsAction()
    {
      // ...

      $response->setSharedMaxAge(60);
    }

With ESI, the full page cache will be valid for 600 seconds, but the news
component cache will only last for 60 seconds.

A requirement of ESI, however, is that the embedded action be accessible
via a URL so the gateway cache can fetch it independently of the rest of
the page. Of course, an action can't be accessed via a URL unless it has
a route that points to it. Symfony2 takes care of this via a generic route
and controller. For the ESI include tag to work properly, you must define
the ``_internal`` route:

.. configuration-block::

    .. code-block:: yaml

        # app/config/routing.yml
        _internal:
            resource: "@FrameworkBundle/Resources/config/routing/internal.xml"
            prefix:   /_internal

    .. code-block:: xml

        <!-- app/config/routing.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>

        <routes xmlns="http://symfony.com/schema/routing"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/routing http://symfony.com/schema/routing/routing-1.0.xsd">

            <import resource="@FrameworkBundle/Resources/config/routing/internal.xml" prefix="/_internal" />
        </routes>

    .. code-block:: php

        // app/config/routing.php
        use Symfony\Component\Routing\RouteCollection;
        use Symfony\Component\Routing\Route;

        $collection->addCollection($loader->import('@FrameworkBundle/Resources/config/routing/internal.xml', '/_internal'));

        return $collection;

.. tip::

    Since this route allows all actions to be accessed via a URL, you might
    want to protect it by using the Symfony2 firewall feature (by allowing
    access to your reverse proxy's IP range). See the :ref:`Securing by IP<book-security-securing-ip>` 
    section of the :doc:`Security Chapter </book/security>` for more information 
    on how to do this.

One great advantage of this caching strategy is that you can make your
application as dynamic as needed and at the same time, hit the application as
little as possible.

.. note::

    Once you start using ESI, remember to always use the ``s-maxage``
    directive instead of ``max-age``. As the browser only ever receives the
    aggregated resource, it is not aware of the sub-components, and so it will
    obey the ``max-age`` directive and cache the entire page. And you don't
    want that.

The ``render`` helper supports two other useful options:

* ``alt``: used as the ``alt`` attribute on the ESI tag, which allows you
  to specify an alternative URL to be used if the ``src`` cannot be found;

* ``ignore_errors``: if set to true, an ``onerror`` attribute will be added
  to the ESI with a value of ``continue`` indicating that, in the event of
  a failure, the gateway cache will simply remove the ESI tag silently.

.. index::
    single: Cache; Invalidation

.. _http-cache-invalidation:

Cache Invalidation
------------------

    "There are only two hard things in Computer Science: cache invalidation
    and naming things." --Phil Karlton

You should never need to invalidate cached data because invalidation is already
taken into account natively in the HTTP cache models. If you use validation,
you never need to invalidate anything by definition; and if you use expiration
and need to invalidate a resource, it means that you set the expires date
too far away in the future.

.. note::

    Since invalidation is a topic specific to each type of reverse proxy,
    if you don't worry about invalidation, you can switch between reverse
    proxies without changing anything in your application code.

Actually, all reverse proxies provide ways to purge cached data, but you
should avoid them as much as possible. The most standard way is to purge the
cache for a given URL by requesting it with the special ``PURGE`` HTTP method.

Here is how you can configure the Symfony2 reverse proxy to support the
``PURGE`` HTTP method::

    // app/AppCache.php

    use Symfony\Bundle\FrameworkBundle\HttpCache\HttpCache;

    class AppCache extends HttpCache
    {
        protected function invalidate(Request $request)
        {
            if ('PURGE' !== $request->getMethod()) {
                return parent::invalidate($request);
            }

            $response = new Response();
            if (!$this->getStore()->purge($request->getUri())) {
                $response->setStatusCode(404, 'Not purged');
            } else {
                $response->setStatusCode(200, 'Purged');
            }

            return $response;
        }
    }

.. caution::

    You must protect the ``PURGE`` HTTP method somehow to avoid random people
    purging your cached data.

Summary
-------

Symfony2 was designed to follow the proven rules of the road: HTTP. Caching
is no exception. Mastering the Symfony2 cache system means becoming familiar
with the HTTP cache models and using them effectively. This means that, instead
of relying only on Symfony2 documentation and code examples, you have access
to a world of knowledge related to HTTP caching and gateway caches such as
Varnish.

Learn more from the Cookbook
----------------------------

* :doc:`/cookbook/cache/varnish`

.. _`Things Caches Do`: http://tomayko.com/writings/things-caches-do
.. _`Cache Tutorial`: http://www.mnot.net/cache_docs/
.. _`Varnish`: http://www.varnish-cache.org/
.. _`Reverse proxy mode da Squid`: http://wiki.squid-cache.org/SquidFaq/ReverseProxy
.. _`expiration model`: http://tools.ietf.org/html/rfc2616#section-13.2
.. _`validation model`: http://tools.ietf.org/html/rfc2616#section-13.3
.. _`RFC 2616`: http://tools.ietf.org/html/rfc2616
.. _`HTTP Bis`: http://tools.ietf.org/wg/httpbis/
.. _`P4 - Conditional Requests`: http://tools.ietf.org/html/draft-ietf-httpbis-p4-conditional-12
.. _`P6 - Caching: Browser and intermediary caches`: http://tools.ietf.org/html/draft-ietf-httpbis-p6-cache-12
.. _`ESI`: http://www.w3.org/TR/esi-lang
