Product selectors
=================

An algorithm states which data products it receives with one
``phlex::product_selector`` per input, passed to ``input_family(...)``:

.. code-block:: cpp

   m.transform("fit", fit, concurrency::unlimited)
     .input_family(product_selector{.creator = "finder", .layer = "event", .suffix = "seeds"},
                   product_selector{.creator = "geometry", .layer = "job"});

Every field is optional, and an unset field matches any value.
The product type is not written in the selector: Phlex takes it from the
type of the corresponding algorithm parameter.
Types are compared by their structure rather than by name, so two distinct
C++ types can be treated as the same; set ``suffix`` (or ``creator``) when
products of different types could otherwise match the same selector.

Fields
------

``creator``
   Matches either the *label* of the plugin that created the product (its
   key in the ``modules`` table) or the name of the algorithm that created
   it.
   For products of explicit providers and sources, it matches the creator
   name the provider declared.

``layer``
   The data layer of the product.
   See `When the layer is required`_ below.

``suffix``
   The suffix given to ``output_product_suffixes(...)`` by the creating
   algorithm.
   An algorithm that does not call ``output_product_suffixes(...)`` creates
   its products with the empty suffix, so a suffix is only needed to tell
   apart several products of the same type from one creator.

``stage``
   The stage that created the product; ``"CURRENT"`` means the job's own
   stage.
   Products from earlier stages can only come from providers.

An empty string is not accepted for ``creator`` or ``layer``.

When the layer is required
--------------------------

The layer may be omitted only for a single-input transform, observer or
predicate.
The selector then matches the product in whichever layer it is created;
for a product supplied by a provider, the provider's layer is used.
For algorithms with more than one input, and for every fold and unfold,
each selector must name a layer; otherwise registration fails with
``Must specify layers in the product selectors for node ...``.

Products are delivered only from the layer the selector names.
For products created by algorithms in the job, the layer is checked as
products arrive, not when the graph is built.
A selector that names the wrong layer therefore does not cause an error;
the algorithm is simply never called.
(If no algorithm or provider can create a selected product at all, the job
fails with ``No provider found for the following required products``.)
If an algorithm unexpectedly does not run, check its layers, and run with
``SPDLOG_LEVEL=debug`` to see how its inputs were connected.

Ambiguous selectors
-------------------

If a selector matches products that differ in suffix, creator or type,
graph construction fails with ``Ambiguous product selector -- see errors``.
The preceding log messages list the candidates and say which field to add.
Matching products that differ only in their layer are allowed.

Selectors in configuration files
--------------------------------

A selector can be read directly from a plugin's configuration, where it is
an object with any of the keys ``creator``, ``layer``, ``suffix`` and
``stage``:

.. code-block:: jsonnet

   {
     modules: {
       fit: {
         cpp: 'track_fit',
         seeds: { creator: 'finder', layer: 'event', suffix: 'seeds' },
         geometry: { creator: 'geometry', layer: 'job' },
       },
     },
   }

.. code-block:: cpp

   PHLEX_REGISTER_ALGORITHMS(m, config)
   {
     m.transform("fit", fit, phlex::concurrency::unlimited)
       .input_family(config.get<phlex::product_selector>("seeds"),
                     config.get<phlex::product_selector>("geometry"));
   }

A JSON array of selector objects can be read as
``std::array<phlex::product_selector, N>``.
Missing keys leave the corresponding fields unset.
Other keys are ignored without an error, so a misspelled key (``suffx``)
silently widens the selector.

Designated initializers
-----------------------

``product_selector`` is meant to be initialized with a subset of its fields.
With ``-Wextra``, GCC and Clang warn about the omitted fields
(``-Wmissing-field-initializers``).
The warning is expected; projects that build with ``-Werror`` may want
``-Wno-missing-field-initializers``.
