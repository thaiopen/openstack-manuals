..
    Warning: Do not edit this file. It is automatically generated from the
    software project's code and your changes will be overwritten.

    The tool to generate this file lives in openstack-doc-tools repository.

    Please make any changes needed in the code, then run the
    autogenerate-config-doc tool from the openstack-doc-tools repository, or
    ask for help on the documentation mailing list, IRC channel or meeting.

.. _cinder-keymgr:

.. list-table:: Description of key manager configuration options
   :header-rows: 1
   :class: config-ref-table

   * - Configuration option = Default value
     - Description
   * - **[keymgr]**
     -
   * - ``api_class`` = ``cinder.keymgr.conf_key_mgr.ConfKeyManager``
     - (String) The full class name of the key manager API class
   * - ``encryption_api_url`` = ``http://localhost:9311/v1``
     - (String) Url for encryption service.
   * - ``encryption_auth_url`` = ``http://localhost:5000/v3``
     - (String) Authentication url for encryption service.
   * - ``fixed_key`` = ``None``
     - (String) Fixed key returned by key manager, specified in hex
