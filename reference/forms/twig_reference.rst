.. index::
   single: Forms; Twig form function reference

Twig �ablonlar� Form Fonksiyonlar�
=====================================

Formlar� olu�tururken kullan�labilecek olan t�m Twig fonksiyonlar� bu belgede verilmi�tir.
Birka� de�i�ik fonksiyon bulunmaktad�r ve her biri formun farkl� k�s�mlar�n�n (etiketler, 
hatalar, ara�lar.. vs.) olu�turulmas�ndan sorumludur.

form_label(form.name, label, variables)
---------------------------------------

Verilen alan i�in etiketi olu�turur. �zel olarak g�r�nmesini istedi�iniz bir etiketi
opsiyonel olan ikinci parametrede verebilirsiniz.

.. code-block:: jinja

    {{ form_label(form.name) }}

    {# A�a��daki iki �rnek de yap�sal olarak do�rudur #}
    {{ form_label(form.name, 'Your Name', { 'attr': {'class': 'foo'} }) }}
    {{ form_label(form.name, null, { 'label': 'Your name', 'attr': {'class': 'foo'} }) }}

form_errors(form.name)
----------------------

Verilan alan i�in hatalar� olu�turur.

.. code-block:: jinja

    {{ form_errors(form.name) }}

    {# herhangi "genel" bir hatay� g�ster #}
    {{ form_errors(form) }}

form_widget(form.name, variables)
---------------------------------

Verilan alan i�in HTML ara� ��kt�s�n� olu�turur. E�er b�t�n bir forma ve ya form alanlar� toplulu�una
uygulan�rsa, i�erdi�i her bir form sat�r� olu�turulur.

.. code-block:: jinja

    {# bir kod blo�unu "foo" s�n�f�n� ekleyerek olu�turur #}
    {{ form_widget(form.name, { 'attr': {'class': 'foo'} }) }}

``form_widget`` i�in belirtilen ikinci parametre bir de�i�kenler dizisidir. 
En �ok kullan�lan de�i�ken ``attr`` dir, HTML arac�na uygulanacak HTML 
�zelliklerini i�erir. Baz� durumlarda, baz� tiplere ait �ablonla ilgili 
ba�ka �zellikler de verilebilir. Bunlar her bir tip i�in ayr�ca a��klanm��t�r.

form_row(form.name, variables)
------------------------------

Renders the "row" of a given field, which is the combination of the field's
label, errors and widget.

.. code-block:: jinja

    {# render a field row, but display a label with text "foo" #}
    {{ form_row(form.name, { 'label': 'foo' }) }}

The second argument to ``form_row`` is an array of variables. The templates
provided in Symfony only allow to override the label as shown in the example
above.

form_rest(form, variables)
--------------------------

This renders all fields that have not yet been rendered for the given form.
It's a good idea to always have this somewhere inside your form as it'll
render hidden fields for you and make any fields you forgot to render more
obvious (since it'll render the field for you).

.. code-block:: jinja

    {{ form_rest(form) }}

form_enctype(form)
------------------

If the form contains at least one file upload field, this will render the
required ``enctype="multipart/form-data"`` form attribute. It's always a
good idea to include this in your form tag:

.. code-block:: html+jinja

    <form action="{{ path('form_submit') }}" method="post" {{ form_enctype(form) }}>