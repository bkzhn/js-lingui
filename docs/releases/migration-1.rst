********************************
Migration guide from 0.x to 1.x
********************************

Even though only a few projects use LinguiJS_ at the moment, the purpose of
this guide is to setup a high-quality migration path for future major versions.

Newline normalization in Babel plugins
=======================================

Formatting of the source code shouldn't affect generated messages.
Both Babel plugins will normalize newlines inside translations.

In Javascript, multiple spaces before and after newline are replaced with
single space, while newlines are removed completely:

.. code-block:: js

   i18n.t`
     This message is wrapped
     on multiple
     lines
  `

   // babel-plugin-lingui-transform-js 0.x
   "This message is wrapped\n  on multiple\n  lines"

   // babel-plugin-lingui-transform-js 1.x
   "This message is wrapped on multiple lines"

Newline normalization in React follows JSX conventions: newlines between text
are replaced with single space, but newlines between components or between text
and a component are removed and no space is added:

.. code-block:: jsx

   <Trans>
     This message is wrapped
     on <em>multiple</em>
     <strong>lines</strong>
   </Trans>

   // babel-plugin-lingui-transform-react 0.x
   "This message is wrapped\n  on <em>multiple<em>\n  <0>lines</0>"

   // babel-plugin-lingui-transform-react 1.x
   "This message is wrapped on <0>multiple<0><1>lines</1>"

As weird as it is, this is consistent with how JSX works. If you need to force
a space at the end of a line, add ``{" "}``:

.. code-block:: jsx

   <Trans>
     This message is wrapped
     on <em>multiple</em>{" "}
     <strong>lines</strong>
   </Trans>

   // This will result in:
   "This message is wrapped on <0>multiple<0> <1>lines</1>"

This change won't break the code, but it's necessary to update message catalogs.

Javascript API (``lingui-i18n``)
================================

I18n constructor replaced with factory function
-----------------------------------------------

``I18n`` constructor has been replaced with the ``setupI18n`` factory function:

.. code-block:: js

   import { setupI18n } from 'lingui-i18n'

   // lingui-i18n 0.x
   const i18n = new I18n(
     language: string,
     messages: {[language: string]: Messages}
   )

   // lingui-i18n 1.x
   const i18n = setupI18n({
     language: string,
     catalogs?: {
       {[language: string]}: {
         messages: Messages
       }
     },
     development?: Development
   })

``messages`` were also replaced with ``catalogs``, more info
:ref:`below <migration1-messages-catalogs>`.

``lingui-i18n`` still exports the default instance of ``I18n`` class, but as
a named export:

.. code-block:: js

   // lingui-i18n 0.x
   // also works in lingui-i18n 1.x, but deprecated
   import i18n from 'lingui-i18n'

   // lingui-i18n 1.1
   import { i18n } from 'lingui-i18n'

Explicit development mode
-------------------------

The biggest change in this first major release is support for compiled message
catalogs. Most i18n libraries parse and compile messages on the fly,
which makes them heavy and slow. The compiler is still useful in
development, but now it has to be enabled manually:

.. code-block:: js

   import { setupI18n } from 'lingui-i18n'

   const dev = process.env.NODE_ENV !== 'production'
     // this import is required in development only
     ? require('lingui-i18n/dev')
     : null

   const i18n = setupI18n({
     language: 'en',
     development: dev
   })

Development data includes compiler and plural rules for all languages. Both
are very large and unnecessary in production, because ``lingui-i18n``
supports loading of compiled message catalogs.

.. _migration1-messages-catalogs:

Messages replaced with catalogs
-------------------------------

Plural rules are removed from library completely, because compiled message
catalogs contain language-specific data, including plural rules. ``messages``
were replaced with ``catalogs`` to simplify loading all required data:

In previous version, loading of messages looked like this (``lingui-i18n 1.x``):

.. code-block:: js

   // lingui-i18n 0.x
   type Messages = {
     [messageId: string]: string
   }

   type AllMessages = {
     [language: string]: Messages
   }

   // setupI18n({ messages: AllMessages })
   const i18n = setupI18n({
     messages: {
       en: {
         msg: 'Hello'
       }
     }
   })

   // i18n.load(messages: AllMessages)
   i18n.load({
     en: {
       msg: 'Hello'
     }
   })

Now it looks like this (``lingui-i18n 1.x``):

.. code-block:: js

   type Messages = {
     // support both static and compiled messages
     [messageId: string]: string | Function
   }

   type Catalog = {
     messages: Messages,
     languageData: {
       // required in production
       plurals: Function
     }
   }

   type Catalogs = {
     [language: string]: Catalog
   }

   // setupI18n({ catalogs: Catalogs })
   const i18n = setupI18n({
     catalogs: {
       en: {
         messages: {
           msg: 'Hello'
         }
       }
     }
   })

   // i18n.load(catalogs: Catalogs)
   i18n.load({
     en: {
       messages: {
         msg: 'Hello'
       }
     }
   })

More about [loading message catalogs]().

React API (``lingui-react``)
============================

Changes in React API reflect changes in underlying core ``lingui-i18n``.

``InjectI18n`` was removed and replaced with ``withI18n`` decorator. Loading
``InjectI18n`` in recent versions of ``lingui-react@<1.0.0`` would raise a
deprecation warning in console.

Development data must be loaded explicitly. It works in the same way as in
``lingui-i18n``:

.. code-block:: jsx

   import { I18nProvider } from 'lingui-react'

   const dev = process.env.NODE_ENV !== 'production'
     // this import is required in development only
     ? require('lingui-i18n/dev')
     : null

   const App = () => {
       <I18nProvider language="en" development={dev}>
           <Content />
       </I18nProvider>
   }

Also, messages were replaced with catalogs:

.. code-block:: jsx

   import { I18nProvider } from 'lingui-react'

   const dev = process.env.NODE_ENV !== 'production'
     // this import is required in development only
     ? require('lingui-i18n/dev')
     : null

   const catalog = {
       messages: {
           msg: "Hello"
       }
   }

   const App = () => {
       <I18nProvider language="en" catalogs={{ en: catalog }} development={dev}>
           <Content />
       </I18nProvider>
   }

CLI (lingui-cli)
================

``lingui export`` command now creates a more complex message catalog, which contains
not only translations, but also default messages (if any) and locations from where
the strings were extracted.

Running ``lingui export`` will generate the new-style catalog while merging translations
from the old one. This process is completely seamless. However, any external
tool must be updates to accept the new-style catalogs.
