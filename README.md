# nRF5 SDK for mesh 4.0.0 : LPN friendship SAR Bug

This is a repo to allow for easy reproduction of a bug with LPN friendship.

Reproduction sequence is below.

The original README text from Nordic was moved to [README_original.md](./README_original.md).

## Basic Setup

You will need two nRF52-DK boards:

- LPN node: Generic onoff Client, LPN
- Light Switch Server: Generic onoff Server, Friend Node

1. Install tooling:

    - gcc (this was built with `8.3.1 20190703 (release)`)
    - cmake
    - ninja
    - Nordic SoftDevice (this was built with `s132_7.0.1`)

    Details outside the scope of this document, see
    https://infocenter.nordicsemi.com/index.jsp?topic=%2Fstruct_sdk%2Fstruct%2Fsdk_mesh_latest.html

1. Set up a BT-Mesh provisioner, we use the [BubblyNet App](https://bubblynet.com/app.html)

    - Create a group for publish/subscribe; we will call ours `GroupA`

1. Clone this repo and set up the build

        git clone https://github.com/siriobalmelli/nrf_mesh_4.0.0_lpn_bug.git
        cd nrf_mesh_4.0.0_lpn_bug
        mkdir build && cd build
        cmake -GNinja ..
        ninja nRF5_SDK  # will install nRF52 SDK in the directory above nrf_mesh_4.0.0_lpn_bug
        cmake -GNinja ..

1. Set up the Light Switch Server

        ninja flash_light_switch_server_nrf52832_xxAA_s132_7.0.1

    - Provision the node (keep proxy enabled)
    - Subscribe `Element1` -> `Generic OnOff Server` to `GroupA`

## Reproduce Mesh LPN Friendship SAR Bug

1. Flash the LPN Node

        ninja flash_lpn_nrf52832_xxAA_s132_7.0.1

    - Access device logging with `JLinkRTTLogger`

1. Provision the LPN node (disable proxy).
Log output:

        <t:          0>, main.c,  552, ----- BLE Mesh LPN Demo -----
        <t:      16397>, main.c,  503, Initializing and adding models
        <t:      21177>, mesh_app_utils.c,   65, Device UUID (raw): D9A0B9D5070BC14591784504A8F0A2B3
        <t:      21181>, mesh_app_utils.c,   70, Device UUID : D5B9A0D9-0B07-45C1-9178-4504A8F0A2B3
        <t:    7061513>, ble_softdevice_support.c,  104, Successfully updated connection parameters
        <t:    7609388>, main.c,  485, NRF_MESH_EVT_DISABLED
        <t:    7609960>, main.c,  141, Successfully provisioned
        <t:    7609966>, main.c,  154, Node Address: 0x000E 
        <t:    7656000>, ble_softdevice_support.c,  104, Successfully updated connection parameters
        <t:    8715234>, config_server.c,  629, dsm_appkey_add(appkey_handle:0 appkey_index:0)
        <t:    8731421>, config_server.c, 2433, Access  Info:
                element_index=0     model_id = 2-FFFF       model_handle=1
        <t:    8747640>, config_server.c, 2433, Access  Info:
                element_index=1     model_id = 1001-FFFF        model_handle=2

1. Press `Button 3` to have LPN node establish a friendship with the Light Switch Server.
Log output:

        <t:     972433>, main.c,  337, Button 2 pressed
        <t:     972435>, main.c,  277, Initiating the friendship establishment procedure.
        <t:     975843>, main.c,  408, Received friend offer from 0x000D
        <t:     979201>, main.c,  455, Friendship established with: 0x000D
        <t:     989283>, main.c,  437, Friend poll procedure complete

1. Use the app to subscribe LPN `Element 2` -> `Generic OnOff Client` to `GroupA`.
Log output:

        <t:    6294774>, main.c,  437, Friend poll procedure complete
        <t:    6617330>, main.c,  214, Publication set
        <t:    6622322>, main.c,  437, Friend poll procedure complete
        <t:    6665867>, main.c,  214, Publication set
        <t:    6670889>, main.c,  437, Friend poll procedure complete
        <t:    6703776>, main.c,  437, Friend poll procedure complete
        <t:    6736678>, main.c,  437, Friend poll procedure complete
        <t:    6769564>, main.c,  437, Friend poll procedure complete
        <t:    6802464>, main.c,  493, NRF_MESH_EVT_SAR_FAILED
        <t:    6802468>, main.c,  437, Friend poll procedure complete
        <t:    7121461>, main.c,  214, Publication set
        <t:    7126455>, main.c,  437, Friend poll procedure complete
        <t:    7157858>, main.c,  437, Friend poll procedure complete
        <t:    7190770>, main.c,  437, Friend poll procedure complete
        <t:    7223668>, main.c,  437, Friend poll procedure complete
        <t:    7256578>, main.c,  437, Friend poll procedure complete
        <t:    7296514>, main.c,  493, NRF_MESH_EVT_SAR_FAILED
        <t:    7296517>, main.c,  437, Friend poll procedure complete
        <t:    7611998>, main.c,  214, Publication set
        <t:    7617019>, main.c,  437, Friend poll procedure complete
        <t:    7644907>, main.c,  437, Friend poll procedure complete
        <t:    7677804>, main.c,  437, Friend poll procedure complete
        <t:    7710693>, main.c,  437, Friend poll procedure complete
        <t:    7743593>, main.c,  437, Friend poll procedure complete
        <t:    7776481>, main.c,  493, NRF_MESH_EVT_SAR_FAILED
        <t:    7776485>, main.c,  437, Friend poll procedure complete

    As you can see, LPN receives a publication-set 4 times, but the mesh
    stack throws `NRF_MESH_EVT_SAR_FAILED` on reply.

    The app never receives a reply and eventually times out.
