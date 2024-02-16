Penjelasan Kode Program APERD Dealer Desktop
=======


Mutual Fund Screen
-------

penjelasan bagian layar ini itu apa....


Penjelasan mengambil client dulu.....


dibawah ini ngejelasan setiap tab nya gitu....

Product List
~~~~~~~

Menampilkan seluruh reksadana yang tersedia dan yang dimiliki oleh client. Sebelum menampilkan daftar reksadana harus memilih client terlebih dahulu.





Portfolio  List
~~~~~~~



Transaction History
~~~~~~~



Subscription
-------

Pada bagian subscription ini semua proses berada pada class ``Subsciption``, yang disimpan pada file :file:`subscription.kt`. Proses yang akan dijelaskan dari menambah *amount* reksadana sampai *submit*/*order* reksadana.


Add Product
~~~~~~~
Pertama, untuk dapat memasukkan jumlah nominal reksadana yang ingin dibeli dengan menggunakan fungsi ``addProduct()``

.. code-block:: kotlin

    class Subscription : Fragment("${AppProperties.appName} - Dealer Subscription Screen")  {
        // other code...
        private fun addProduct() {
            val product = vm.productProperty.value
            val nav = product.nav.toDoubleOrNull() ?: 0.0

            val maxCashOnHand = eligibleCashOnHand.value.replace(",", "").toDouble()

            val validator = Validator()
                .rule(vm.amount > 0, "Amount must be greater than 0.")
                .rule(nav.roundToLong() > 0L, "Nav/Unit must be greater than 0.")
                .rule(vm.grandTotal.value.toDouble() <= maxCashOnHand,
                    "Grand total must not be more than Rp ${eligibleCashOnHand.value}."
                )
                .rule(
                    vm.amount >= product.minSubs.toLong(),
                    "Minimum amount of ${product.fundName} is " +
                          "${Formatter.numberFormat(product.minSubs.toLong())}."
                )
                .rule(
                    !vm.accName.value.isNullOrBlank(),
                    "Sorry, there is no Account Name."
                )
                .rule(
                    !vm.rdnNumber.value.isNullOrBlank(),
                    "Sorry, there is no RDN Number."
                )
                .rule(
                    !vm.bankAcc.value.isNullOrBlank(),
                    "Sorry, there is no Bank Account."
                )
                .validate()


            if (!validator.isValid()) {
                Alerts.warning(validator.getErrorMessages().joinToString(separator = "\n"))
                return
            }

            frgLoader.openModal(
                stageStyle = StageStyle.TRANSPARENT,
                modality = Modality.APPLICATION_MODAL,
                escapeClosesWindow = false,
                resizable = false,
                owner = this@Subscription.currentWindow
            )

            val taskCB = runAsync { WebServiceData.custodianBankByFundCode(product.fundCode) }

            taskCB.setOnSucceeded {
                val cb = taskCB.value

                if (cb != null) {
                    custodianBanks.setAll(cb)

                    val rdnCode = Constant.bankInfo[userProfile.rdnBnCode]?.get("bank_code") ?: ""
                    val cbCode = custodianBanks[0].bankCode
                    val amount = vm.amount.toString()

                    val taskBC = runAsync {
                        WebServiceData.bankCharge(rdnCode, cbCode, amount)
                    }

                    taskBC.setOnSucceeded {
                        val bc = taskBC.value

                        if (bc != null) {
                            bankChargeItem = bc
                        }

                        addProductToTable()
                        updateSummaryTotalSection()

                        resetInputs()

                        frgLoader.close()
                    }

                    taskBC.setOnFailed {
                        val exception = taskBC.exception

                        frgLoader.close()
                        Alerts.errors("Bank Charge: " + exception.message)
                    }
                } else {
                    Alerts.errors("Failed to add ${product.fundName} fund, please try again later")
                }
            }

            taskCB.setOnFailed {
                val exception = taskCB.exception

                frgLoader.close()
                Alerts.errors("Custodian Bank: " + exception.message)
            }
        }
    }


Menentukan beberapa *variable* yaitu product, nav, dan maxCashOnHand

.. code-block::  kotlin

    val product = vm.productProperty.value
    val nav = product.nav.toDoubleOrNull() ?: 0.0

    val maxCashOnHand = eligibleCashOnHand.value.replace(",", "").toDouble()



Melakukan validasi terlebih dahulu, jika tidak valid maka proses akan diberhentikan.

.. code-block:: kotlin

    val validator = Validator()
        .rule(vm.amount > 0, "Amount must be greater than 0.")
        .rule(nav.roundToLong() > 0L, "Nav/Unit must be greater than 0.")
        .rule(vm.grandTotal.value.toDouble() <= maxCashOnHand,
            "Grand total must not be more than Rp ${eligibleCashOnHand.value}."
        )
        .rule(
            vm.amount >= product.minSubs.toLong(),
            "Minimum amount of ${product.fundName} is " +
                  "${Formatter.numberFormat(product.minSubs.toLong())}."
        )
        .rule(
            !vm.accName.value.isNullOrBlank(),
            "Sorry, there is no Account Name."
        )
        .rule(
            !vm.rdnNumber.value.isNullOrBlank(),
            "Sorry, there is no RDN Number."
        )
        .rule(
            !vm.bankAcc.value.isNullOrBlank(),
            "Sorry, there is no Bank Account."
        )
        .validate()


    if (!validator.isValid()) {
        Alerts.warning(validator.getErrorMessages().joinToString(separator = "\n"))
        return
    }


Menampilkan *loader indicator* pada layar.

.. code-block:: kotlin

    frgLoader.openModal(
        stageStyle = StageStyle.TRANSPARENT,
        modality = Modality.APPLICATION_MODAL,
        escapeClosesWindow = false,
        resizable = false,
        owner = this@Subscription.currentWindow
    )


Proses penambahan jumlah nominal diawali dengan mengambil data *custodian bank* terlebih dahulu.

.. code-block:: kotlin

    val taskCB = runAsync { WebServiceData.custodianBankByFundCode(product.fundCode) }


Kalau gagal akan menampilkan *alert errors* dan *loader indicator* dihilangkan ``frgLoader.close()``.

.. code-block:: kotlin

    taskCB.setOnFailed {
        val exception = taskCB.exception

        frgLoader.close()
        Alerts.errors("Custodian Bank: " + exception.message)
    }

Jika sukses mengambil data *custodian bank*, akan dilanjutkan untuk mengambil data *bank charge*. Sebelum mengambil data *bank charge* harus dicek *null* tidak nya. Jika berhasil proses berlanjut dan kalau gagal akan menampilkan pesan error pada layar.

.. code-block:: kotlin

    taskCB.setOnSucceeded {
        val cb = taskCB.value

        if (cb != null) {
            custodianBanks.setAll(cb)

            val rdnCode = Constant.bankInfo[userProfile.rdnBnCode]?.get("bank_code") ?: ""
            val cbCode = custodianBanks[0].bankCode
            val amount = vm.amount.toString()

            val taskBC = runAsync {
                WebServiceData.bankCharge(rdnCode, cbCode, amount)
            }
            // handle error and success bank charge data...
        } else {
            Alerts.errors("Failed to add ${product.fundName} fund, please try again later")
        }
    }


Kalau gagal mengambil data *bank charge* akan menampilkan *alert errors* dan *loader indicator* dihilangkan.

.. code-block:: kotlin

    taskBC.setOnFailed {
        val exception = taskBC.exception

        frgLoader.close()
        Alerts.errors("Bank Charge: " + exception.message)
    }


Setelah berhasil mengambil data *bank charge*, harus dicek terlebih dahulu. Kalau tidak ``null`` akan disimpan pada variable ``bankChargeItem``, supaya data *bank charge* dapat disimpan. Selanjutnya, reksadana akan disimpan dengan menggunakan *method* ``addProductToTable()``, dan juga *summary section* akan diperbaharui dengan ``updateSummaryTotalSection()``. Tidak lupa *form* direset ``resetInputs()`` dan *loader indicator* di hilangkan.

.. code-block:: kotlin

    taskBC.setOnSucceeded {
        val bc = taskBC.value

        if (bc != null) {
            bankChargeItem = bc
        }

        addProductToTable()
        updateSummaryTotalSection()

        resetInputs()

        frgLoader.close()
    }


Fungsi AddProductToTable()
*******

Fungsi ini digunakan untuk menyimpan reksadana yang sudah ditambahkan pada tabel, atau dalam variable ``fundOrders``. Sebelum disimpan data harus dicek terlebih dahulu apakah sudah tersedia atau belum. Jika sudah ada, data tidak akan ditambahkan melainkan hanya memperbaharui jumlah *amount* (``amount_lama`` + ``amount_baru``), *bank charge* dan *custodian bank*. Jika tidak ada, maka data akan ditambahkan pada tabel.

.. code-block:: kotlin

    class Subscription : Fragment("${AppProperties.appName} - Dealer Subscription Screen")  {
        // other code...
        private fun addProductToTable() {
            val product = vm.productProperty.value
            val feeSubs = product.feeSubs.toDoubleOrNull() ?: 0.0

            val dataToUpdate = fundOrders.find { it.fundCode == product.fundCode }
            vm.amount += dataToUpdate?.amount ?: 0L

            val data = FundOrderSubs(
                fundCode = product.fundCode,
                fundName = product.fundName,
                lastPrice = product.nav.toDouble(),
                amount = vm.amount,
                trxFee = feeSubs,
                unit = vm.amount.toDouble() / product.nav.toDouble(),
                dealerFee = 0.0,
                bankChargeItem = bankChargeItem
            )

            if (!custodianBanks.isEmpty()) {
                data.cb = custodianBanks[0]
            }

            if (dataToUpdate != null) {
                fundOrders[fundOrders.indexOf(dataToUpdate)] = data
            } else {
                fundOrders.add(data)
            }
        }
    }



Fungsi updateSummaryTotalSection()
*******

Selanjutnya fungsi ``updateSummaryTotalSection()`` berguna untuk memperbaharui *summary section* pada layar subscription.

.. code-block:: kotlin

    class Subscription : Fragment("${AppProperties.appName} - Dealer Subscription Screen")  {
        // other code...
        private fun updateSummaryTotalSection() {
            vm.totalAmount.value = fundOrders.sumByLong { it.amount }

            vm.totalTrxFee.value = fundOrders.map { (it.trxFee * it.amount.toDouble()) / 100 }
                .sumByLong { it.toLong() }

            vm.totalDealerFee.value = fundOrders.map { (it.dealerFee * it.amount.toDouble()) / 100 }
                .sumByLong { it.toLong() }

            vm.totalBc.value = fundOrders.map { it.bankChargeItem.bankCharge.toDoubleOrNull() ?: 0.0 }
                .sumOf { it.roundToLong() }

            vm.grandTotal.value = vm.totalAmount.value + vm.totalTrxFee.value + vm.totalDealerFee.value + vm.totalBc.value
        }
    }


Fungsi resetInputs()
*******

Terakhir fungsi ``resetInputs()`` berguna agar *input amount* dapat direset.

.. code-block:: kotlin

    class Subscription : Fragment("${AppProperties.appName} - Dealer Subscription Screen")  {
        // other code...
        private fun resetInputs() {
            vm.amount = 0
        }
    }

Subscribe Product
~~~~~~~
Proses *subscribe* dilakukan dengan menekan tombol *submit* dan akan mengekseskusi *function* ``order()``.

.. code-block:: kotlin

    class Subscription : Fragment("${AppProperties.appName} - Dealer Subscription Screen")  {
        // other code...
        private fun order() {
            val maxCashOnHand = eligibleCashOnHand.value.replace(",", "").toDouble()

            val validator = Validator()
                .rule(GlobalState.clientsAperdState.selectedClient != null, Constant.NO_CLIENT_SELECTED)
                .rule(!fundOrders.isEmpty(), "Please select a product.")
                .rule(vm.grandTotal.value.toDouble() <= maxCashOnHand,
                    "Grand total must not be more than Rp ${eligibleCashOnHand.value}."
                )
                .rule(
                    !vm.accName.value.isNullOrBlank(),
                    "Sorry, there is no Account Name."
                )
                .rule(
                    !vm.rdnNumber.value.isNullOrBlank(),
                    "Sorry, there is no RDN Number."
                )
                .rule(
                    !vm.bankAcc.value.isNullOrBlank(),
                    "Sorry, there is no Bank Account."
                )
                .validate()

            if (!validator.isValid()) {
                Alerts.warning(validator.getErrorMessages().joinToString(separator = "\n"))
                return
            }

            setMutualFundOrders()

            alert(Alert.AlertType.CONFIRMATION, "", "Are you sure you want to subscribe this mutual fund?", ButtonType.YES, ButtonType.CANCEL, title = "Order Confirmation") {
                if (it == ButtonType.YES) {
                    frgLoader.openModal(
                        stageStyle = StageStyle.TRANSPARENT,
                        modality = Modality.APPLICATION_MODAL,
                        escapeClosesWindow = false,
                        resizable = false,
                        owner = this@Subscription.currentWindow
                    )

                    val task = runAsync { WebServiceData.subscribe(subscribeProducts) }

                    task.setOnSucceeded {
                        frgLoader.close()

                        Alerts.information("Successfully subscribe mutual fund")
                        currentStage?.close()
                    }

                    task.setOnFailed {
                        frgLoader.close()

                        val exception = task.exception
                        Alerts.errors("Subscription: " + exception.message)
                    }
                }
            }
        }
    }


Menentukan *local variable* untuk menyimpan maximum cash on hand.

.. code-block:: kotlin

    val maxCashOnHand = eligibleCashOnHand.value.replace(",", "").toDouble()


Setelah itu akan dilakukan validasi untuk pengecekan apakah data yang mau dikirim sudah sesuai apa belum.

.. code-block:: kotlin

    val validator = Validator()
        .rule(GlobalState.clientsAperdState.selectedClient != null, Constant.NO_CLIENT_SELECTED)
        .rule(!fundOrders.isEmpty(), "Please select a product.")
        .rule(vm.grandTotal.value.toDouble() <= maxCashOnHand,
            "Grand total must not be more than Rp ${eligibleCashOnHand.value}."
        )
        .rule(
            !vm.accName.value.isNullOrBlank(),
            "Sorry, there is no Account Name."
        )
        .rule(
            !vm.rdnNumber.value.isNullOrBlank(),
            "Sorry, there is no RDN Number."
        )
        .rule(
            !vm.bankAcc.value.isNullOrBlank(),
            "Sorry, there is no Bank Account."
        )
        .validate()

    if (!validator.isValid()) {
        Alerts.warning(validator.getErrorMessages().joinToString(separator = "\n"))
        return
    }



Menyimpan semua data untuk dikirim yang berada pada *function* ``setMutualFundOrders()``.


Menampilkan sebuah pesan konfirmasi sebelum melakukan pembelian reksdana. Jika user menekan tombol *Yes* proses *subscribe product* akan dilakukan.

.. code-block:: kotlin

    alert(Alert.AlertType.CONFIRMATION, "", "Are you sure you want to subscribe this mutual fund?", ButtonType.YES, ButtonType.CANCEL, title = "Order Confirmation") {
        if (it == ButtonType.YES) {
            // code for handle subscribe product...
        }
    }


Menampilkan *loader indicator* pada layar.

.. code-block:: kotlin

    frgLoader.openModal(
        stageStyle = StageStyle.TRANSPARENT,
        modality = Modality.APPLICATION_MODAL,
        escapeClosesWindow = false,
        resizable = false,
        owner = this@Subscription.currentWindow
    )


Selanjutnya, request untuk *subscribe product*

.. code-block:: kotlin

    val task = runAsync { WebServiceData.subscribe(subscribeProducts) }


Jika berhasil, *loader indicator* akan dihilangkan dan menampilkan pesan *success*. Setelah user *click* tombol *oke* atau *close*, layar subscription akan ditutup dengan ``currentStage?.close()``.

.. code-block:: kotlin

    task.setOnSucceeded {
        frgLoader.close()

        Alerts.information("Successfully subscribe mutual fund")
        currentStage?.close()
    }


Jika gagal, *loader indicator* akan dihilangkan dan menampilkan pesan *error* pada layar.

.. code-block:: kotlin

    task.setOnFailed {
        frgLoader.close()

        val exception = task.exception
        Alerts.errors("Subscription: " + exception.message)
    }



Fungsi setMutualFundOrders()
*******

Pada fungsi ini akan melakukan penyimapan data pada variabel ``fundOrders`` ke ``subscribeProducts``. Sebelum pemindahan dilakukan data ``subscribeProducts`` akan dihapus terlebih dahulu ``subscribeProducts.clear()``.

.. code-block:: kotlin

    class Subscription : Fragment("${AppProperties.appName} - Dealer Subscription Screen")  {
        // other code...
        private fun setMutualFundOrders() {
            subscribeProducts.clear()
            fundOrders.forEach { fundOrder ->
                val cb = fundOrder.cb
                val bankChargeItem = fundOrder.bankChargeItem

                val rdnBankCode = Constant.bankInfo[userProfile.rdnBnCode]?.get("bank_code") ?: ""
                val trxFeeNominal = (fundOrder.amount.toDouble() * fundOrder.trxFee) / 100
                val dealerFeeNominal = (fundOrder.amount.toDouble() * fundOrder.dealerFee) / 100

                val subscribe = MutualFundOrder(
                    transDate = DateAndTime.now(),
                    transType = Constant.TRANS_TYPE_SUBS,
                    fundCode = fundOrder.fundCode,
                    sid = userProfile.sid,
                    qtyAmount = fundOrder.amount.toString(),
                    qtyUnit = fundOrder.unit.toString(),
                    lastNav = fundOrder.lastPrice.toString(),
                    feeNominal = trxFeeNominal.roundToLong().toString(),
                    feePersen = fundOrder.trxFee.toString(),
                    redmPaymentAccSeqCode = "",
                    redmPaymentBicCode = "",
                    redmPaymentAccNo = "",
                    rdnAccNo = userProfile.rdncbAccNo,
                    rdnBankCode = rdnBankCode,
                    rdnBankName = userProfile.rdncbAccName ?: "",
                    cbAccNo = cb.cbAccNo,
                    cbBankCode = cb.bankCode,
                    cbBankName = cb.cbName,
                    paymentDate = DateAndTime.now(),
                    transferType = Constant.TRANS_TYPE_SUBS,
                    transactionType = bankChargeItem.transactionType,
                    bankCharge = bankChargeItem.bankCharge,
                    deviceId = Constant.DEVICE_ID_DESKTOP,
                    feeNominalDealer = dealerFeeNominal.roundToLong().toString(),
                    feePersenDealer = fundOrder.dealerFee.toString(),
                    dealerName = GlobalState.session.userId,
                )

                subscribeProducts.add(subscribe)
            }
        }
    }


Redemption
-------



Switching
-------




Bulk Order
-------



Bulk Order History
-------



.. autosummary::
   :toctree: generated