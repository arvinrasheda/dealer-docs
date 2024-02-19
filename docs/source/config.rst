Configuration
===================================

Config Env App
----------------
Config untuk pengaturan pada app , ada pada ``path\to\app\var\app.env`` didalam sini kita dapat melakukan untuk log error

.. code-block:: console

    # DEBUG | TRACE
    LOG_LEVEL=ERROR
    ALT_LOG_LEVEL=ERROR
    ALT_WRITE_LOG=true


Config Env Server
----------------
Config yang digunakan untuk modul APERD Pada file Env Server, untuk file sendiri ada dalam folder ``path\to\app\var\config``
berikut ini merupakan contoh dari file .env

.. code-block:: console

    # For Web Socket
    WS_MARKETUPD_URL=wss://dev-hthbg.bahana.co.id:5050
    WS_TRX_URL=wss://dev-hthbg.bahana.co.id:12000

    # interval between reconnect attempts, in seconds
    RECONNECT_INTERVAL=3

    # verbose debugging ws transaction
    DBG_WS_TRX=true
    # verbose debugging ws market update
    DBG_WS_MARKETUPD=true

Config untuk endpoint api APERD (default: https://dev-hthbg.bahana.co.id:12000 ):

.. code-block:: console

    API_URL=https://dev-hthbg.bahana.co.id:12000

Config untuk disable dan enable Dealer Fee APERD (default: false ) pada env server:

.. code-block:: console

    EDITABLE_DEALER_FEE=true

Config untuk update package online
----------------
arahkan uri update ``app/DX-TRADE.cfg``  ke host online yang menyimpan data  ``update``

.. code-block:: console

    --uri=file:////{path}
    uri=http://192.168.10.45/dxtrade/updater/latest/

Config untuk update package offline
----------------

arahkan uri update ``app/DX-TRADE.cfg``   ke folder   ``update`` hasil download:

.. code-block:: console

    --uri=file:////{path}
    uri=file:////C:/Users/user/Documents/Work/dxtrade/update


.. autosummary::
   :toctree: generated

   lumache
