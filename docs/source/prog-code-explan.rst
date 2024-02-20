Penjelasan Kode Program APERD Dealer Desktop
=======


*Mutual Fund Screen*
-------

penjelasan bagian layar ini itu apa....


Penjelasan mengambil client dulu.....


dibawah ini ngejelasan setiap tab nya gitu....

*Product List*
~~~~~~~


*Portfolio  List*
~~~~~~~


*Transaction History*
~~~~~~~


*Subscription*
-------

Pada bagian subscription ini semua proses berada pada class ``Subsciption``, yang disimpan pada
file :file:`subscription.kt`. Proses yang akan dijelaskan dari menambah *amount* reksadana (*add mutual fund*)
sampai *submit*/*order* reksadana (*subscribe mutual fund*).


*Add Mutual Fund*
~~~~~~~
*Function* yang digukanan untuk memasukkan *amount* reksadana yang ingin dibeli ialah menggunakan ``addProduct()``

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


Bagian pertama yaitu menentukan beberapa *variable* diantarannya ``product``, ``nav``, dan ``maxCashOnHand``

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


Kalau gagal akan menampilkan pesan *errors* dan *loader indicator* dihilangkan ``frgLoader.close()``.

.. code-block:: kotlin

    taskCB.setOnFailed {
        val exception = taskCB.exception

        frgLoader.close()
        Alerts.errors("Custodian Bank: " + exception.message)
    }

Jika sukses mengambil data *custodian bank*, akan dilanjutkan untuk mengambil data *bank charge*.
Sebelum mengambil data *bank charge* harus dicek *null* tidak nya *custodian bank*.
Jika berhasil, proses dilanjutkan dan kalau gagal akan menampilkan pesan *error* pada layar.

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


Setelah berhasil mengambil data *bank charge*, harus dicek terlebih dahulu. Kalau tidak ``null`` akan disimpan pada
variable ``bankChargeItem``, supaya data *bank charge* dapat disimpan. Selanjutnya, reksadana akan disimpan dengan
menggunakan *method* ``addProductToTable()``, dan juga *summary section* akan diperbaharui dengan
``updateSummaryTotalSection()``. Tidak lupa *form* direset ``resetInputs()`` dan *loader indicator* di hilangkan.

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


Semua *function* yang berada pada ``taskBC.setOnSucceeded {...}`` akan dijelaskan dengan detail sebagai berikut:

#. *Function AddProductToTable()*
    Fungsi ini digunakan untuk menyimpan reksadana yang sudah ditambahkan pada tabel, atau dalam variable ``fundOrders``.
    Sebelum disimpan data harus dicek terlebih dahulu apakah sudah tersedia atau belum. Jika sudah ada, data tidak akan
    ditambahkan melainkan hanya memperbaharui jumlah *amount* (``amount_lama`` + ``amount_baru``), *bank charge* dan
    *custodian bank*. Jika tidak ada, maka data akan ditambahkan pada tabel.

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


#. *Function updateSummaryTotalSection()*
    Selanjutnya fungsi ``updateSummaryTotalSection()`` berguna untuk memperbaharui *summary section* pada layar
    subscription.

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


#. *Function resetInputs()*
    Terakhir fungsi ``resetInputs()`` berguna agar *input amount* direset ke 0.

    .. code-block:: kotlin

        class Subscription : Fragment("${AppProperties.appName} - Dealer Subscription Screen")  {
            // other code...
            private fun resetInputs() {
                vm.amount = 0
            }
        }

*Subscribe Mutual Fund*
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

            alert(Alert.AlertType.CONFIRMATION, "", "Are you sure you want to subscribe this mutual fund?",
                ButtonType.YES, ButtonType.CANCEL, title = "Order Confirmation"
            ) {
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


Menyimpan semua data untuk dikirim yang berada pada *function* ``setMutualFundOrders()``. Pada fungsi ini akan
melakukan penyimapan data pada variabel ``fundOrders`` ke ``subscribeProducts``. Sebelum pemindahan dilakukan,
data ``subscribeProducts`` akan dihapus terlebih dahulu ``subscribeProducts.clear()``.

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


Menampilkan sebuah pesan konfirmasi sebelum melakukan pembelian reksdana. Jika user menekan tombol *Yes*
proses *subscribe product* akan dilakukan.

.. code-block:: kotlin

    alert(Alert.AlertType.CONFIRMATION, "", "Are you sure you want to subscribe this mutual fund?",
        ButtonType.YES, ButtonType.CANCEL, title = "Order Confirmation"
    ) {
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


Selanjutnya, *request* untuk *subscribe product*

.. code-block:: kotlin

    val task = runAsync { WebServiceData.subscribe(subscribeProducts) }


Jika berhasil, *loader indicator* akan dihilangkan dan menampilkan pesan *success*.
Setelah user *click* tombol *oke* atau *close*, layar *subscription* akan ditutup dengan ``currentStage?.close()``.

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


*Redemption*
-------

Untuk *redemption* semua proses berada pada *Redemption Class*, yang disimpan pada file ``redemption.kt``.
Penjelasan setiap proses akan dimulai dari tambah reksadana (*add mutual fund*) sampai
*submit*/*redeem* reksadana (*redeem mutual fund*).

*Add Mutual Fund*
~~~~~~~

Proses menambahkan reksadana untuk dijual berada pada *function* ``addPortofolio()``, yang dapat dilihat pada *code*
dibawah ini.

.. code-block:: kotlin

    class Redemption : Fragment("${AppProperties.appName} - Dealer Redemption Screen") {
        //other code...
        private fun addPortofolio() {
            val portofolioItem = vm.portofolioProperty.value
            val maxUnit = portofolioItem.unit.toDouble()
            val maxUnitScaled = Formatter.setScale(maxUnit, Constant.DIGIT_COMMA_UNIT).toDouble()
            val maxAmount = MutualFundUtils.getMaxPortofolioAmount(portofolioItem).roundToLong()
            val minRedeem = portofolioItem.minRedem.toLong()

            val dataToUpdate = fundOrders.find { it.fundCode == portofolioItem.fundCode }

            val insertedUnit: Double
            val insertedAmount: Long
            if (dataToUpdate !== null) {
                insertedUnit = dataToUpdate.unit.toDouble() + vm.unitProperty.value
                insertedAmount = dataToUpdate.amount.toLong() + vm.amountProperty.value
            } else {
                insertedUnit = vm.unitProperty.value
                insertedAmount = vm.amountProperty.value
            }

            val validator = Validator()
                .rule(vm.unitProperty > 0.0, "Unit must grater than 0.0.")
                .rule(vm.amountProperty > 0, "Amount must grater than 0.")
                .rule(
                    vm.amountProperty >= minRedeem,
                    "Minimum amount of ${vm.portofolioProperty.value.fundName} is ${Formatter.numberFormat(minRedeem)}."
                )
                .rule(insertedUnit <= maxUnitScaled, "Maximum unit is ${Formatter.numberFormat(maxUnitScaled, Constant.DIGIT_COMMA_UNIT)} unit.")
                .rule(insertedAmount <= maxAmount, "Amount must not grater than ${Formatter.numberFormat(maxAmount)}.")
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
                owner = this@Redemption.currentWindow
            )

            val task = runAsync {
                WebServiceData.fundByFundCode(portofolioItem.fundCode)
            }

            task.setOnSucceeded {
                product = task.value!!

                addPortofolioToTable()
                updateSummaryTotalSection()

                resetInputs()
                frgLoader.close()
            }

            task.setOnFailed {
                frgLoader.close()

                val exception = task.exception
                Alerts.errors("Mutual Fund By Fund Code: " + exception.message)
            }
        }
    }


Pertama, menentukan beberapa *local variables* untuk digunakan nanti pada proses selanjutnya.

.. code-block:: kotlin

    val portofolioItem = vm.portofolioProperty.value
    val maxUnit = portofolioItem.unit.toDouble()
    val maxUnitScaled = Formatter.setScale(maxUnit, Constant.DIGIT_COMMA_UNIT).toDouble()
    val maxAmount = MutualFundUtils.getMaxPortofolioAmount(portofolioItem).roundToLong()
    val minRedeem = portofolioItem.minRedem.toLong()

    val dataToUpdate = fundOrders.find { it.fundCode == portofolioItem.fundCode }

    val insertedUnit: Double
    val insertedAmount: Long


Melakukan pengecekan apakah reksadana sudah tersimpan atau belum, agar dapat menentukan berapa banyak *unit* dan *amount*.

.. code-block:: kotlin

    if (dataToUpdate !== null) {
        insertedUnit = dataToUpdate.unit.toDouble() + vm.unitProperty.value
        insertedAmount = dataToUpdate.amount.toLong() + vm.amountProperty.value
    } else {
        insertedUnit = vm.unitProperty.value
        insertedAmount = vm.amountProperty.value
    }


Melakukan validasi untuk mengecek apakah data yang dimasukkan sudah sesuai atau belum.
Kalau belum proses ini akan diberhentikan.

.. code-block:: kotlin

    val validator = Validator()
        .rule(vm.unitProperty > 0.0, "Unit must grater than 0.0.")
        .rule(vm.amountProperty > 0, "Amount must grater than 0.")
        .rule(
            vm.amountProperty >= minRedeem,
            "Minimum amount of ${vm.portofolioProperty.value.fundName} is ${Formatter.numberFormat(minRedeem)}."
        )
        .rule(insertedUnit <= maxUnitScaled, "Maximum unit is ${Formatter.numberFormat(maxUnitScaled, Constant.DIGIT_COMMA_UNIT)} unit.")
        .rule(insertedAmount <= maxAmount, "Amount must not grater than ${Formatter.numberFormat(maxAmount)}.")
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
        owner = this@Redemption.currentWindow
    )


Mengambil data *mutual fund* untuk mengambil data *fee redemption*.

.. code-block:: kotlin

    val task = runAsync {
        WebServiceData.fundByFundCode(portofolioItem.fundCode)
    }


Jika gagal mengambil data *mutual fund* akan menampilkan pesan *error*
dan *loader indicator* dihilangkan ``frgLoader.close()``.

.. code-block:: kotlin

    task.setOnFailed {
        frgLoader.close()

        val exception = task.exception
        Alerts.errors("Mutual Fund By Fund Code: " + exception.message)
    }


Kalau berhasil mengambil data *mutual fund* maka akan menyimpan reksadana pada tabel dan memperbaharui *summary section*.
Setelah itu lanjut mereset *form input* sama *loader indicator* akan dihilangkan.

.. code-block:: kotlin

    task.setOnSucceeded {
        product = task.value!!

        addPortofolioToTable()
        updateSummaryTotalSection()

        resetInputs()
        frgLoader.close()
    }


Berikut merupakan penjelasan setiap *function* yang berada pada *block* ``task.setOnSucceeded {...}``:

#. *Function addPortofolioToTable()*
    *Function* ini berguna untuk menambahkan data reksadana yang mau dijual pada tabel. Data yang berada pada tabel
    disimpan di *variable* ``fundOrders``. Proses penambahan data ini tidak langsung disimpan begitu saja, tetapi
    melewati pengecekan terlebih dahulu.
    Apakah reksadana sudah ada yang disimpan atau belum, seperti pada *code* ``if (dataToUpdate != null) {}``.
    Jika reksadana kosong akan langsung disimpan, kalau tidak akan menghitung (``unit_lama`` + ``unit_baru``) dan
    (``amount_lama`` + ``amount_baru``) untuk *update* reksadana nya.


    .. code-block:: kotlin

        class Redemption : Fragment("${AppProperties.appName} - Dealer Redemption Screen") {
            //other code...
            private fun addPortofolioToTable() {
                val portofolioItem = vm.portofolioProperty.value

                val unit: Double = vm.unitProperty.value
                val amount: Long = vm.amountProperty.value
                val feeRedm = product.feeRedm

                val data = FundOrderRedm(
                    fundCode =  portofolioItem.fundCode,
                    fundName = portofolioItem.fundName,
                    lastPrice = portofolioItem.lastPrice.toDouble(),
                    trxFee = feeRedm.toDouble(),
                    unit = unit,
                    amount = amount,
                    dealerFee = 0.0
                )

                val dataToUpdate = fundOrders.find { it.fundCode == portofolioItem.fundCode }

                if (dataToUpdate != null) {
                    data.unit = dataToUpdate.unit + unit
                    data.amount = dataToUpdate.amount + amount

                    fundOrders[fundOrders.indexOf(dataToUpdate)] = data
                } else {
                    fundOrders.add(data)
                }
            }
        }


#. *Function updateSummaryTotalSection()*
    Memiliki fungsi untuk memperbaharui *summary section*

    .. code-block:: kotlin

        class Redemption : Fragment("${AppProperties.appName} - Dealer Redemption Screen") {
            //other code...
            private fun updateSummaryTotalSection() {
                vm.totalAmount.value = fundOrders.sumOf { it.amount.toLong() }

                vm.totalTrxFee.value = fundOrders.map { (it.trxFee.toDouble() * it.amount.toDouble()) / 100 }
                    .sumByLong { it.toLong() }

                vm.totalDealerFee.value = fundOrders.map { (it.dealerFee.toDouble() * it.amount.toDouble()) / 100 }
                    .sumByLong { it.toLong() }

                vm.estTotalRedm.value = vm.totalAmount.value - vm.totalTrxFee.value - vm.totalDealerFee.value
            }
        }


#. *Function resetInputs()*
    Berfungsi untuk *reset* *input unit* dan *amount* menjadi 0.

    .. code-block:: kotlin

        class Redemption : Fragment("${AppProperties.appName} - Dealer Redemption Screen") {
            //other code...
            private fun resetInputs() {
                vm.unitProperty.set(0.0)
                vm.amountProperty.set(0L)
            }
        }


*Redeem Mutual Fund*
~~~~~~~

Proses penjualan reksadana dilakukan setelah tombol *submit* ditekan, dan akan menjalankan *function* ``redeem()``.

.. code-block:: kotlin

    class Redemption : Fragment("${AppProperties.appName} - Dealer Redemption Screen") {
        //other code...
        private fun redeem() {
            val validator = Validator()
                .rule(GlobalState.clientsAperdState.selectedClient != null, Constant.NO_CLIENT_SELECTED)
                .rule(!fundOrders.isEmpty(), "Please add portfolio.")
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

            alert(
                Alert.AlertType.CONFIRMATION, "",
                "Are you sure you want to redeem this mutual fund?", ButtonType.YES,
                ButtonType.CANCEL, title = "Order Confirmation"
            ) {
                if (it == ButtonType.YES) {
                    frgLoader.openModal(
                        stageStyle = StageStyle.TRANSPARENT,
                        modality = Modality.APPLICATION_MODAL,
                        escapeClosesWindow = false,
                        resizable = false,
                        owner = this@Redemption.currentWindow
                    )

                    val task = runAsync { WebServiceData.redeem(redeemProducts) }

                    task.setOnSucceeded {
                        frgLoader.close()

                        Alerts.information("Successfully redeem mutual fund")
                        currentStage?.close()
                    }

                    task.setOnFailed {
                        frgLoader.close()

                        val exception = task.exception
                        Alerts.errors(exception.message)
                    }
                }
            }
        }
    }


Pertama yang dilakukan adalah validasi data apakah sudah sesuai atau belum.

.. code-block:: kotlin

    val validator = Validator()
        .rule(GlobalState.clientsAperdState.selectedClient != null, Constant.NO_CLIENT_SELECTED)
        .rule(!fundOrders.isEmpty(), "Please add portfolio.")
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


Menyimpan semua data untuk dikirim yang berada pada *function* ``setMutualFundOrders()``. Pada fungsi ini akan
melakukan penyimpanan data pada variabel ``fundOrders`` ke ``redeemProducts``. Sebelum pemindahan dilakukan,
data ``redeemProducts`` akan dihapus terlebih dahulu ``redeemProducts.clear()``.

.. code-block:: kotlin

    class Redemption : Fragment("${AppProperties.appName} - Dealer Redemption Screen") {
        //other code...
        private fun setMutualFundOrders() {
            redeemProducts.clear()
            fundOrders.forEach { fundOrder ->
                val trxFeeNominal = (fundOrder.amount.toDouble() * fundOrder.trxFee.toDouble()) / 100
                val dealerFeeNominal = (fundOrder.amount.toDouble() * fundOrder.dealerFee.toDouble()) / 100

                val redeem = MutualFundOrder(
                    transDate = DateAndTime.now(),
                    transType = Constant.TRANS_TYPE_REDM,
                    fundCode = fundOrder.fundCode,
                    sid = userProfile.sid,
                    qtyAmount = fundOrder.amount.toString(),
                    qtyUnit = fundOrder.unit.toString(),
                    lastNav = fundOrder.lastPrice.toString(),
                    feeNominal = trxFeeNominal.roundToLong().toString(),
                    feePersen = fundOrder.trxFee.toString(),
                    feeNominalDealer = dealerFeeNominal.roundToLong().toString(),
                    feePersenDealer = fundOrder.dealerFee.toString(),
                    redmPaymentAccSeqCode = "",
                    redmPaymentBicCode = "",
                    redmPaymentAccNo = "",
                    paymentDate = DateAndTime.now(),
                    transferType = Constant.TRANS_TYPE_REDM,
                    deviceId = Constant.DEVICE_ID_DESKTOP,
                    dealerName = GlobalState.session.userId
                )

                redeemProducts.add(redeem)
            }
        }
    }


Menampilkan sebuah pesan konfirmasi sebelum melakukan penjualan reksadana. Jika user menekan tombol *Yes*
proses *redemption* akan dilakukan.

.. code-block:: kotlin

    alert(
        Alert.AlertType.CONFIRMATION, "",
        "Are you sure you want to redeem this mutual fund?", ButtonType.YES,
        ButtonType.CANCEL, title = "Order Confirmation"
    ) {
        if (it == ButtonType.YES) {
            // code for handle redemption...
        }
    }


Menampilkan *loader indicator* pada layar.

.. code-block:: kotlin

    frgLoader.openModal(
        stageStyle = StageStyle.TRANSPARENT,
        modality = Modality.APPLICATION_MODAL,
        escapeClosesWindow = false,
        resizable = false,
        owner = this@Redemption.currentWindow
    )


Selanjutnya, *request* untuk *redemption*

.. code-block:: kotlin

    val task = runAsync { WebServiceData.redeem(redeemProducts) }


Jika berhasil, *loader indicator* akan dihilangkan dan menampilkan pesan *success*.
Setelah user *click* tombol *oke* atau *close*, layar *redemption* akan ditutup dengan ``currentStage?.close()``.

.. code-block:: kotlin

    task.setOnSucceeded {
        frgLoader.close()

        Alerts.information("Successfully redeem mutual fund")
        currentStage?.close()
    }


Jika gagal, *loader indicator* akan dihilangkan dan menampilkan pesan *error* pada layar.

.. code-block:: kotlin

    task.setOnFailed {
        frgLoader.close()

        val exception = task.exception
        Alerts.errors(exception.message)
    }


*Switching*
-------

*Switching* berfungsi untuk memindahkan reksadana yang dimiliki kepada reksadana yang lain.
Pada *Switching* semua proses berada pada *Switching Class*, yang disimpan pada file ``switching.kt``.
Proses yang akan dijelaskan meliputi pemilihan *switching in*,
tambah *switching in* (*add switching in*),
dan *submit*/*switch* reksadana (*switch mutual fund*).

*Choose Product To Switching In*
~~~~~~~

Memilih *Switching In Mutual Fund* dilakukan dengan menekan nama reksadana yang berada pada daftar reksadana dibawah
*form add switching out*. Proses ini dilakukan dengan menggunakan *function* ``choosenProduct(fundCode: String)``,
yang ditampilkan dibawah ini.

.. code-block:: kotlin

    class Switching : Fragment("${AppProperties.appName} - Dealer Switching Screen")  {
        //other code...
        private fun chosenProduct(fundCode: String) {
            var added = true

            if (
                productIn.value.fundCode.isNotBlank() &&
                productIn.value.fundCode !== fundCode
            ) {
                added = false
                alert(
                    Alert.AlertType.CONFIRMATION, "",
                    "Are you sure you want to change your portfolio?", ButtonType.YES,
                    ButtonType.CANCEL, title = "Confirmation"
                ) {
                    if (it == ButtonType.YES) {
                        added = true
                        resetForm()
                    }
                }
            }

            if (added) {
                productIn.value = list.first { it.fundCode == fundCode }
            }
        }
    }


Pada fungsi di atas memiliki beberapa kondisi untuk menampilkan pesan konfirmasi penggantian data *switching in* , jika mau diganti.
Kedua, kondisi untuk menambahkan data *switching in* yang telah dipilih.


*Add Switching In*
~~~~~~~

Menentukan besaran nominal *amount* atau *unit* reksadana (*switching out*) yang mau dipindahkan dengan
menekan tombol *Add*, lalu akan menjalankan *code* yang berada pada *function* ``addSwitchingIn()``

.. code-block:: kotlin

    class Switching : Fragment("${AppProperties.appName} - Dealer Switching Screen")  {
        //other code...
        private fun addSwitchingIn() {
            val productIn = productIn.value

            val portofolioItem = vm.portofolioProperty.value
            val maxUnit = portofolioItem.unit.toDouble()
            val maxUnitScaled = Formatter.setScale(maxUnit, Constant.DIGIT_COMMA_UNIT).toDouble()
            val maxAmount = MutualFundUtils.getMaxPortofolioAmount(portofolioItem).roundToLong()

            var minSubs = productIn.minSubs.toLongOrNull()
            if (minSubs == null) {
                minSubs = 0L
            }

            val nav = productIn.nav.toDoubleOrNull() ?: 0.0
            val validator = Validator()
                .rule(vm.unitProperty > 0.0, "Unit must grater than 0.")
                .rule(vm.amountProperty > 0L, "Amount must grater than 0.")
                .rule(vm.unitProperty.value <= maxUnitScaled, "Maximum unit is ${Formatter.numberFormat(maxUnitScaled)} unit.")
                .rule(vm.amountProperty.value <= maxAmount, "Amount must not grater than ${Formatter.numberFormat(maxAmount)}.")
                .rule(productIn.fundCode.isNotBlank(), "Please select product.")
                .rule(productIn.fundCode.isBlank() || nav.roundToLong() > 0L, "Nav/Unit must be greater than 0.")
                .rule(vm.amountProperty > minSubs, "Minimum transaction for ${productIn.fundName} is ${Formatter.numberFormat(minSubs)}.")
                .validate()

            if (!validator.isValid()) {
                Alerts.warning(validator.getErrorMessages().joinToString(separator = "\n"))
                return
            }

            vm.switchingInTrxFee.value = productIn.feeSwitch.toDoubleOrNull() ?: 0.0

            productInNotSelected.set(false)
            updateSummaryTotalSection()
        }
    }


Hal pertama yang dilakukan yaitu menentukan beberapa *local vaiables* terlebih dahulu.

.. code-block:: kotlin

    val productIn = productIn.value

    val portofolioItem = vm.portofolioProperty.value
    val maxUnit = portofolioItem.unit.toDouble()
    val maxUnitScaled = Formatter.setScale(maxUnit, Constant.DIGIT_COMMA_UNIT).toDouble()
    val maxAmount = MutualFundUtils.getMaxPortofolioAmount(portofolioItem).roundToLong()

    var minSubs = productIn.minSubs.toLongOrNull()


Selanjutnya, melakukan pengecekan pada ``minSubs`` apakah ``null`` atau tidak.
Jika ``null`` maka nilai dari ``minSubs`` akan diganti menjadi 0.

.. code-block:: kotlin

    if (minSubs == null) {
        minSubs = 0L
    }


Melakukan validasi agar data yang mau ditambahkan dicek terlebih dahulu benar atau tidak nya.

.. code-block:: kotlin

    val nav = productIn.nav.toDoubleOrNull() ?: 0.0
    val validator = Validator()
        .rule(vm.unitProperty > 0.0, "Unit must grater than 0.")
        .rule(vm.amountProperty > 0L, "Amount must grater than 0.")
        .rule(vm.unitProperty.value <= maxUnitScaled, "Maximum unit is ${Formatter.numberFormat(maxUnitScaled)} unit.")
        .rule(vm.amountProperty.value <= maxAmount, "Amount must not grater than ${Formatter.numberFormat(maxAmount)}.")
        .rule(productIn.fundCode.isNotBlank(), "Please select product.")
        .rule(productIn.fundCode.isBlank() || nav.roundToLong() > 0L, "Nav/Unit must be greater than 0.")
        .rule(vm.amountProperty > minSubs, "Minimum transaction for ${productIn.fundName} is ${Formatter.numberFormat(minSubs)}.")
        .validate()

    if (!validator.isValid()) {
        Alerts.warning(validator.getErrorMessages().joinToString(separator = "\n"))
        return
    }


Selanjutnya menyimpan data *switching in transaction fee*

.. code-block:: kotlin

    vm.switchingInTrxFee.value = productIn.feeSwitch.toDoubleOrNull() ?: 0.0


*Variable* ``productInNotSelected`` isi dengan *false*

.. code-block:: kotlin

    productInNotSelected.set(false)



Terakhir, perbaharui *summary section* dengan menggunakan *function* ``updateSummaryTotalSection()``.

.. code-block:: kotlin

    class Switching : Fragment("${AppProperties.appName} - Dealer Switching Screen")  {
        //other code...
        private fun updateSummaryTotalSection() {
            val totalTrxFee =  vm.switchingInTrxFee.value * vm.amountProperty.value.toDouble() / 100
            vm.totalTrxFee.value = totalTrxFee.toLong()

            val totalDealerFee = vm.dealerFee.value * vm.amountProperty.value.toDouble() / 100
            vm.totalDealerFee.value = totalDealerFee.toLong()

            updateUnit(vm.unitProperty.value, vm.amountProperty.value)
            updateAmount(vm.amountProperty.value)
        }

        private fun addSwitchingIn() {
            //other code..
            updateSummaryTotalSection()
        }
    }


*Switch Mutual Fund*
~~~~~~~

Melakukan *switch* reksadana dengan menekan tombol *submit*, dan akan menjalankan *function* ``switch()``.

.. code-block:: kotlin

    class Switching : Fragment("${AppProperties.appName} - Dealer Switching Screen")  {
        //other code...
        private fun switch() {
            val validator = Validator()
                .rule(vm.portofolioProperty.value.fundCode.isNotBlank(), "Sorry, no mutual fund is willing to exchange.")
                .rule(productIn.value.fundCode.isNotBlank(), "Please select product.")
                .rule(vm.unitTransfer.value > 0.0, "Unit transfer must grater than 0.")
                .rule(vm.switchingOutAmount.value > 0L, "Switching out amount must grater than 0.")
                .rule(vm.estUnitsEarned > 0.0, "Est. units earned must grater than 0.")
                .rule(vm.estSwitchingInAmount > 0L, "Est. switching in amount must grater than 0.")
                .rule(!userProfile.userId.isNullOrBlank(), "Sorry, there is no user profile for client name ${client.fullName}.")
                .validate()

            if (!validator.isValid()) {
                Alerts.warning(validator.getErrorMessages().joinToString(separator = "\n"))
                return
            }

            setMutualFundOrderForSwitch()

            alert(
                Alert.AlertType.CONFIRMATION, "",
                "Are you sure you want to switch this mutual fund?", ButtonType.YES,
                ButtonType.CANCEL, title = "Order Confirmation"
            ) {
                if (it == ButtonType.YES) {
                    frgLoader.openModal(
                        stageStyle = StageStyle.TRANSPARENT,
                        modality = Modality.APPLICATION_MODAL,
                        escapeClosesWindow = false,
                        resizable = false,
                        owner = this@Switching.currentWindow
                    )

                    val task = runAsync { WebServiceData.switch(switchProducts) }

                    task.setOnSucceeded {
                        frgLoader.close()

                        Alerts.information("Successfully switch mutual fund")
                        currentStage?.close()
                    }

                    task.setOnFailed {
                        frgLoader.close()

                        val exception = task.exception
                        Alerts.errors(exception.message)
                    }
                }
            }
        }
    }


Pertama yang dilakukan adalah validasi data apakah sudah sesuai atau belum.

.. code-block:: kotlin

    val validator = Validator()
        .rule(vm.portofolioProperty.value.fundCode.isNotBlank(), "Sorry, no mutual fund is willing to exchange.")
        .rule(productIn.value.fundCode.isNotBlank(), "Please select product.")
        .rule(vm.unitTransfer.value > 0.0, "Unit transfer must grater than 0.")
        .rule(vm.switchingOutAmount.value > 0L, "Switching out amount must grater than 0.")
        .rule(vm.estUnitsEarned > 0.0, "Est. units earned must grater than 0.")
        .rule(vm.estSwitchingInAmount > 0L, "Est. switching in amount must grater than 0.")
        .rule(!userProfile.userId.isNullOrBlank(), "Sorry, there is no user profile for client name ${client.fullName}.")
        .validate()

    if (!validator.isValid()) {
        Alerts.warning(validator.getErrorMessages().joinToString(separator = "\n"))
        return
    }


Menyimpan semua data untuk dikirim yang berada pada *function* ``setMutualFundOrderForSwitch()``. Pada fungsi ini akan
melakukan penyimpanan data pada variabel ``switchProducts``. Sebelum penyimpanan dilakukan,
data ``switchProducts`` akan dihapus terlebih dahulu ``switchProducts.clear()``.

.. code-block:: kotlin

    class Redemption : Fragment("${AppProperties.appName} - Dealer Redemption Screen") {
        //other code...
        private fun setMutualFundOrderForSwitch() {
            switchProducts.clear()
            val trxFeeNominalIn = (vm.amountProperty.value.toDouble() * vm.switchingInTrxFee.value) / 100
            val dealerFeeNominal = (vm.amountProperty.value.toDouble() * vm.dealerFee.value) / 100

            val switch = MutualFundOrder(
                transDate = DateAndTime.now(),
                transType = Constant.TRANS_TYPE_SWTC,
                fundCodeOut = vm.productProperty.value.fundCode,
                sid = userProfile.sid,
                qtyAmountOut = qtyAmountOut,
                qtyUnitOut = vm.unitTransfer.value.toString(),
                lastNav = productIn.value.nav,
                feeNominal = trxFeeNominalIn.roundToLong().toString(),
                feePersen = vm.switchingInTrxFee.value,
                feeNominalDealer = dealerFeeNominal.roundToLong().toString(),
                feePersenDealer = vm.dealerFee.value.toString(),
                switchingFeeCharge = productIn.value.feeSwitch,
                paymentDate = DateAndTime.now(),
                transferType = Constant.TRANS_TYPE_SWTC,
                fundCodeIn = productIn.value.fundCode,
                deviceId = Constant.DEVICE_ID_DESKTOP,
                dealerName = GlobalState.session.userId
            )

            switchProducts.add(switch)
        }
    }


Menampilkan sebuah pesan konfirmasi sebelum melakukan *switch* reksadana. Jika user menekan tombol *Yes*
proses *switching* akan dilakukan.

.. code-block:: kotlin

    alert(
        Alert.AlertType.CONFIRMATION, "",
        "Are you sure you want to switch this mutual fund?", ButtonType.YES,
        ButtonType.CANCEL, title = "Order Confirmation"
    ) {
        if (it == ButtonType.YES) {
            // code for handle switching...
        }
    }


Menampilkan *loader indicator* pada layar.

.. code-block:: kotlin

    frgLoader.openModal(
        stageStyle = StageStyle.TRANSPARENT,
        modality = Modality.APPLICATION_MODAL,
        escapeClosesWindow = false,
        resizable = false,
        owner = this@Switching.currentWindow
    )


Selanjutnya, *request* untuk *switching*

.. code-block:: kotlin

    val task = runAsync { WebServiceData.switch(switchProducts) }


Jika berhasil, *loader indicator* akan dihilangkan dan menampilkan pesan *success*.
Setelah user *click* tombol *oke* atau *close*, layar *switching* akan ditutup dengan ``currentStage?.close()``.

.. code-block:: kotlin

    task.setOnSucceeded {
        frgLoader.close()

        Alerts.information("Successfully switch mutual fund")
        currentStage?.close()
    }


Jika gagal, *loader indicator* akan dihilangkan dan menampilkan pesan *error* pada layar.

.. code-block:: kotlin

    task.setOnFailed {
        frgLoader.close()

        val exception = task.exception
        Alerts.errors(exception.message)
    }



*Bulk Order*
-------

Bulk order merupakan fitur untuk dapat membeli banyak reksadana (*subscription*), yang berada pada layar
*Bulk Order Subscription Screen*. Semua kode program berada dalam ``BulkSubscription Class``, disimpan pada file
```bulksubscriptionk.kt``. Kode program yang akan dijelaskan dimulai dari *download template*, *upload file*,
*process inquiry data*, dan *execute bulk order*.

*Download Template*
~~~~~~~

Proses *download template* dilakukan dengan menekan tombol *Download Template* dan akan menjalankan *function*
``downloadTemplate()``. *File* yang akan *generate* berformat *.xlsx*.

.. code-block:: kotlin

    class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
        //other code...
        private fun downloadTemplate() {
            val filename = "bulk-order-subscription-${Formatter.getDateAsYMD()}"
            val fileChooser = FileChooser()
            val extension = FileChooser.ExtensionFilter("Microsoft Excel Worksheet", "*.xlsx")
            fileChooser.extensionFilters.add(extension)
            fileChooser.initialFileName = filename
            val file = fileChooser.showSaveDialog(currentWindow)
            if (file != null) {
                writeExcel(file)
            }
        }
    }


Pertama, menentukan beberapa *variables* untuk digunakan pada proses selanjutnya.

.. code-block:: kotlin

    val filename = "bulk-order-subscription-${Formatter.getDateAsYMD()}"
    val fileChooser = FileChooser()
    val extension = FileChooser.ExtensionFilter("Microsoft Excel Worksheet", "*.xlsx")


Menyimpan nama *file* dan *extention file* pada ``FileChooser()`` *Object*,

.. code-block:: kotlin

    fileChooser.extensionFilters.add(extension)
    fileChooser.initialFileName = filename


Proses selanjutnya, menemtukan tempat penyimpanan *file* yang akan diunduh dengan menggunakan
``showSaveDialog(currentWindow)``. *Function* ini akan mengambil lokasi *file download* juga.
Lalu dilakukan pengecekan apakah sudah ditentukan atau belum. Kalau sudah proses *download* akan di eksekusi
``writeExcel(file)``.


.. code-block:: kotlin

    val file = fileChooser.showSaveDialog(currentWindow)
    if (file != null) {
        writeExcel(file)
    }


*Function* ``writeExcel(file)`` masih berada pada *class* yang sama dan berguna untuk membuat *file excel* dengan
template yang sudah ditentukan. *Template* yang dibuat berupa nama beberapa kolom diatarannya *Client Code*,
*IFUA No*, dan *Amount* (Nominal) yang berada pada kode
``columns.setAll("Client Code", "IFUA No", "Fund Code", "Amount (Nominal)")``.

.. code-block:: kotlin

    class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
        //other code...
        private fun writeExcel(file: File) {
            val wb = XSSFWorkbook()
            try {
                val outputStream = FileOutputStream(file)
                val sheet = wb.createSheet("order")
                val columns: ObservableList<String> = observableListOf()

                columns.setAll("Client Code", "IFUA No", "Fund Code", "Amount (Nominal)")

                var colNumber = 0
                val rowHeader = sheet.createRow(0)
                columns.forEach { title ->
                    val cell = rowHeader.createCell(colNumber++)
                    cell.setCellValue(title)
                }
                wb.write(outputStream)
                outputStream.close()
            } catch (e: Exception) {
                e.message?.let { Alerts.errors(it) }
            } finally {
                wb.close()
            }
        }
    }


*Upload File*
~~~~~~~

Fitur *upload file* harus sesuai dengan *template* yang sudah diunduh, dengan format *.xlsx*. Proses ini dilakukan
dengan menekan tombol *Upload* yang akan menjalankan *function* ``loadFileDialog()``.

.. code-block:: kotlin
    class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
        //other code...
        private fun loadFileDialog() {
            try {
                isFileChoosing = true
                updateUploadButtonState()

                val file: List<File> = chooseFile("Select file", arrayOf(
                    FileChooser.ExtensionFilter("Microsoft Excel Worksheet", "*.xlsx")
                ))

                isFileChoosing = false
                updateUploadButtonState()

                if (file.isEmpty()) return
                resetDataAll()

                val selectedFile: File = file.first()
                val filePath = selectedFile.absolutePath
                loadOrdersFromFile(filePath)
            } catch (e: Exception) {
                e.message?.let { Alerts.errors(it) }

                isFileChoosing = false
                updateUploadButtonState()
            }
        }
    }


Langkah awal akan membuka *file explorer* untuk memilih *file* yang akan diunggah. Lalu tombol *Upload* akan *disable*
agar user tidak dapat membuka *file explorer* lagi. Proses ini diawali dengan *disable* tombol *Upload* terlebih dahulu,
setelah itu *file explorer* akan dibuka.

.. code-block:: kotlin

    isFileChoosing = true
    updateUploadButtonState()

    val file: List<File> = chooseFile("Select file", arrayOf(
        FileChooser.ExtensionFilter("Microsoft Excel Worksheet", "*.xlsx")
    ))


Setelah *file* dipilih, tombol *Upload* akan diaktifkan lagi dengan menggunakan kode

.. code-block:: kotlin

    isFileChoosing = false
    updateUploadButtonState()


Selanjutnya dilakukan pengecekan apakah *file* kosong atau tidak. Kalau *file* kosong proses *upload* akan diberhentikan.

.. code-block:: kotlin

    if (file.isEmpty()) return

Jika *file* tidak kosong, selanjutnya  semua data akan direset ``resetDataAll()``.

.. code-block:: kotlin
    class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
        //other code...
        private fun resetDataAll() {
            isCheckedAll.value = false
            isUploaded.value = false
            isProcessInquiryClicked.value = false

            userProfiles.clear()

            totalItems.set(0)
            totalItemsProcessed.set(0)
            totalAmountProcessed.set(0)

            orderBookingList.clear()
            checkedList.clear()
            cashBalances.clear()
        }
    }


Setelah itu, file akan dibaca oleh sistem dan hasilnya akan ditampilkan pada tabel di layar.

.. code-block:: kotlin

    val selectedFile: File = file.first()
    val filePath = selectedFile.absolutePath
    loadOrdersFromFile(filePath)


Pada *Function* ``loadOrdersFromFile(filePath)`` ini akan menyimpan data pada *excel* ke tabel.
Pertama hasil *read file excel* disimpan pada *variable* ```val bulkOrderList = bulkLoader.loadFile(filePath)``.
Setelah berhasil membaca *file* akan dipindahkan pada *variable* ``orderBookingList.setAll(bulkOrder)`` untuk ditampilkan
pada tabel. Setelah pemindahan selesai, *variable* ``isUploaded`` *set* ke *true*, agar tombol *Process Inquiry Data*
dapat diaktifkan.

.. code-block:: kotlin

    class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
        //other code...
        private fun loadOrdersFromFile(filePath: String) {
            isUploaded.value = false

            val bulkLoader = BulkOrderSubscriptionLoader()
            val bulkOrderList = bulkLoader.loadFile(filePath)

            val bulkOrder = bulkOrderList.mapIndexed { idx, order ->
                val trxType = Constant.TRANS_TYPE_SUBS.toInt()

                val unformattedAmount = order.amount.replace(",", "")
                val amount = unformattedAmount.toDoubleOrNull() ?: throw Exception("The amount column must be filled in with numbers only.")

                val numb = idx + 1
                order.orderId = numb.toString()

                order.type = trxType.toString()
                order.amount = amount.roundToLong().toString()

                order.isCheckAble = false

                order
            }

            orderBookingList.setAll(bulkOrder)
            isUploaded.value = true
        }
    }


Terakhir, jika terjadi kesalahan atau *error* akan menampilkan pesan ke layar ``e.message?.let { Alerts.errors(it) }``.
Setelah pesan *error* ditampilkan, tombol *Upload* akan diaktifkan kembali.

.. code-block:: kotlin

    try {
        //handle upload file...
    } catch (e: Exception) {
        e.message?.let { Alerts.errors(it) }

        isFileChoosing = false
        updateUploadButtonState()
    }


*Process Inquiry Data*
~~~~~~~

Fitur ini berfungsi untuk memvalidasi semua data yang sudah diunggah, sebelum semua data di *order*. Proses ini berjalan
seleteh tombol *Process Inquiry Data* ditekan dan akan menjalankan *function* ``processInquiryData()``, seperti pada
kode dibawah ini.

.. code-block:: kotlin

    class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
        //other code...
        private fun processInquiryData() {
            try {
                if (clientList.isEmpty()) {
                    Alerts.warning(
                        "There is no client list data. Please try logging in again then click this button"
                    )
                    return
                }
                frgLoader.openModal(
                    stageStyle = StageStyle.TRANSPARENT,
                    modality = Modality.APPLICATION_MODAL,
                    escapeClosesWindow = false,
                    resizable = false,
                    owner = this@BulkSubscription.currentWindow
                )

                cashBalances.clear()
                unprocessedMsgClearAll()

                runAsync {
                    orderBookingList.map { order ->
                        order.status = AperdOrderStatus.UNPROCESSED

                        val client = clientList.firstOrNull { it.clientCode == order.saCode }
                        if (client == null) {
                            order.unprocessedMsgList.add("Sorry, there is no client data for SA Code ${order.saCode}.")
                        } else {
                            order.name = client.fullName

                            runBlocking {
                                setUserData(client.clientCode)
                            }

                            //need to remove any leading zeroes client code, cuz the response is not use that
                            client.clientCode = Helper.removeLeadingZeros(client.clientCode)

                            val user = userProfiles.find { it.custCode.equals(client.clientCode) }
                            if (user !== null) {
                                val inquiry = InquiryTransaction(
                                    fundCode = order.fundCode,
                                    ifuaNo = order.ifuaNo,
                                    rdnSwiftCode = Constant.bankInfo[user.rdnBnCode]?.get("bank_code") ?: "",
                                    amountNominal = order.amount,
                                    unitNominal = "0",
                                    transactionType = Constant.TRANS_TYPE_SUBS
                                )

                                val inquiryTransaction = runBlocking { WebServiceData.inquiryTransaction(inquiry) }
                                if (inquiryTransaction != null) {
                                    order.sid = inquiryTransaction.sid
                                    order.lastNav = inquiryTransaction.lastNavUnit
                                    order.estUnit = inquiryTransaction.estUnit
                                    order.bankCharge = inquiryTransaction.bankCharge
                                    order.cbAccNo = inquiryTransaction.cbAccNo
                                    order.cbSwiftCode = inquiryTransaction.cbSwiftCode
                                    order.cbBankName = inquiryTransaction.cbAccName
                                    order.trxFee = inquiryTransaction.feePersen
                                    order.trxFeeAmount = inquiryTransaction.feeAmount
                                    order.dealerFee = "0"
                                    order.dealerFeeAmount = "0"
                                    order.transferType = inquiryTransaction.transferType
                                    order.transactionType = inquiryTransaction.transactionType

                                    if (order.amount.toLong() > 0L) {
                                        order.isCheckAble = true
                                    } else {
                                        order.unprocessedMsgList.add("Amount (Nominal) must be more than 0.")
                                    }
                                } else {
                                    order.unprocessedMsgList.add("Failed to retrieve mutual fund data for Fund Code ${order.fundCode}.")
                                }
                            } else {
                                order.unprocessedMsgList.add("Sorry, there is no profile data for ${order.name}.")
                            }

                            val cashBalance = cashBalances.filter { (key, _) -> key == client.clientCode }
                            if (cashBalance.isEmpty()) {
                                order.unprocessedMsgList.add("Sorry, failed to get Eligible Cash On Hand Data. Please try again later.")
                            }
                        }

                        order
                    }
                } ui { bulkOrder ->
                    orderBookingList.setAll(bulkOrder)

                    validateCashOnHand()
                    updateCheckedAllStatus()
                    updateSummaryTotalSection()

                    isProcessInquiryClicked.set(true)

                    frgLoader.close()
                }
            } catch (e: Exception) {
                Platform.runLater {
                    Alerts.errors("Sorry, failed to process inquiry data, please try again later.")

                    e.message?.let {
                        Logger.warning("${tagName}--processInquiryData", e.message!!)
                    }
                }
            }
        }
    }


Pertama, akan melakukan pengecekan data *client* yang sudah diambil setelah berhasil *login* aplikasi.

.. code-block:: kotlin

    if (clientList.isEmpty()) {
        Alerts.warning(
            "There is no client list data. Please try logging in again then click this button"
        )
        return
    }


Menampilkan *loader indicator* pada layar.

.. code-block:: kotlin

    frgLoader.openModal(
        stageStyle = StageStyle.TRANSPARENT,
        modality = Modality.APPLICATION_MODAL,
        escapeClosesWindow = false,
        resizable = false,
        owner = this@BulkSubscription.currentWindow
    )


Setelah itu, akan menghapus data *cash balance* dan *unprocessed messages*

.. code-block:: kotlin

    cashBalances.clear()
    unprocessedMsgClearAll()


Selanjutnya, kita akan melakukan pengecekan pada setiap data yang sudah diunggah. Pengecekan dilakukan satu persatu,
yang berada pada block ``runAsync {...}``. Setelah pengecekan selesai data pada tabel akan diperbaharui.
Pertama, status akan di isi *Unprocesed* dahulu.

.. code-block:: kotlin

    order.status = AperdOrderStatus.UNPROCESSED


Setelah itu, akan dilakukan pengecekan data *client* apakah tersedia atau tidak. Jika tidak ada, pengecekan akan berhenti
dan menyimpan pesan gagal nya juga.

.. code-block:: kotlin

    val client = clientList.firstOrNull { it.clientCode == order.saCode }
    if (client == null) {
        order.unprocessedMsgList.add("Sorry, there is no client data for SA Code ${order.saCode}.")
    } else {
        //other code...
    }


Jika *client* ada, akan menyimpan data *fullname* dan mengambil data *user profile* dan *cash balance*.
Pengambilan dan penyimpanan data-data yang diambil dilakukan pada *function* ``setUserData(client.clientCode)``.

.. code-block:: kotlin
    class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
        //other code...
        fun setUserData(custCode: String) {
            val userProfileList = handleUserProfileService(custCode)
            val cashList = handleCashBalancesService(custCode)

            if (userProfileList.isNotEmpty() && cashList.isNotEmpty()) {
                setUserProfiles(userProfileList[0])
                setCashBalances(cashList[0])
            }
        }

        private fun processInquiryData() {
            //...
                order.name = client.fullName
                runBlocking {
                    setUserData(client.clientCode)
                }
                //...
        }
    }


Selanjutnya, menghapus angka *prefix* pada *clientCode*

.. code-block:: kotlin

    client.clientCode = Helper.removeLeadingZeros(client.clientCode)


Setelah itu, cek apakah *clientCode* yang sudah diunggah sesuai dengan data *User Profiles*. Kalau tidak, pesan *error*
akan ditampilkan. Perlu diingat *variable* ``userProfiles`` terisi setelah proses pengambilan data *user profile*
 sebelumnya berhasil, yang berada pada kode ``setUserData(client.clientCode)``.

.. code-block:: kotlin

    val user = userProfiles.find { it.custCode.equals(client.clientCode) }
    if (user !== null) {
        //other code...
    } else {
        order.unprocessedMsgList.add("Sorry, there is no profile data for ${order.name}.")
    }


Setelah user dicek, akan *request inquiry transaction*.

.. code-block:: kotlin

    val inquiry = InquiryTransaction(
        fundCode = order.fundCode,
        ifuaNo = order.ifuaNo,
        rdnSwiftCode = Constant.bankInfo[user.rdnBnCode]?.get("bank_code") ?: "",
        amountNominal = order.amount,
        unitNominal = "0",
        transactionType = Constant.TRANS_TYPE_SUBS
    )

    val inquiryTransaction = runBlocking { WebServiceData.inquiryTransaction(inquiry) }


Kode selanjutnya, akan menyimpan data *inquiry transaction* yang sudah diambil, dan jika gagal akan menampilkan pesan
*error*.

.. code-block:: kotlin

    if (inquiryTransaction != null) {
        order.sid = inquiryTransaction.sid
        order.lastNav = inquiryTransaction.lastNavUnit
        order.estUnit = inquiryTransaction.estUnit
        order.bankCharge = inquiryTransaction.bankCharge
        order.cbAccNo = inquiryTransaction.cbAccNo
        order.cbSwiftCode = inquiryTransaction.cbSwiftCode
        order.cbBankName = inquiryTransaction.cbAccName
        order.trxFee = inquiryTransaction.feePersen
        order.trxFeeAmount = inquiryTransaction.feeAmount
        order.dealerFee = "0"
        order.dealerFeeAmount = "0"
        order.transferType = inquiryTransaction.transferType
        order.transactionType = inquiryTransaction.transactionType

        if (order.amount.toLong() > 0L) {
            order.isCheckAble = true
        } else {
            order.unprocessedMsgList.add("Amount (Nominal) must be more than 0.")
        }
    } else {
        order.unprocessedMsgList.add("Failed to retrieve mutual fund data for Fund Code ${order.fundCode}.")
    }


Terakhir, kode untuk menyimpan pembaharuan data ``orderBookingList``.

.. code-block:: kotlin
    orderBookingList.map { order ->
        //other code...
        order
    }


Setelah semua pengecekan data selesai, lanjutkan proses pada *block* ``ui { bulkOrder -> ...}``.

.. code-block:: kotlin

    runAsync {
        //other code...
    } ui { bulkOrder ->
        orderBookingList.setAll(bulkOrder)

        validateCashOnHand()
        updateCheckedAllStatus()
        updateSummaryTotalSection()

        isProcessInquiryClicked.set(true)

        frgLoader.close()
    }


Pertama, *update* data ``orderBookingList`` yang sudah diubah sebelumnya ``orderBookingList.setAll(bulkOrder)``.
Lalu, memvalidasi *cash on hand* dengan menggunakan *function* ``validateCashOnHand()``. *Update checkbox* setiap baris,
lalu *update summary total section*. Tidak lupa untuk mengganti *value* dari ``isProcessInquiryClicked`` menjadi *true,
dan *loader indicator* dihilangkan.


Berikut merupakan penjelasan lebih detail mengenai beberapa *functions* yang berda pada *block* ``ui { bulkOrder -> ...}``.

#. *Function validateCashOnHand()*
    Fungsi ini digunakan agar dapat memvalidasi *cash on hand* untuk setiap *client* dari setiap reksadana yang akan dibeli.
    Mengenai apakah jumah *cash* yang dimiliki *client* memadai untuk membeli reksadana atau tidak. Fungsi ini juga
    yang akan mengganti status dari *Unprocessed* menjadi *Ready for Processing*, agar reksadana dapat dibeli.

    .. code-block:: kotlin

        class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
            //other code...
            private fun validateCashOnHand() {
                cashBalances.forEach { (saCode, cashBalance) ->
                    var cash = cashBalance

                    orderBookingList
                        .filter {
                            val saCodeWithNoLeadingZeros = Helper.removeLeadingZeros(it.saCode)
                            saCodeWithNoLeadingZeros == saCode
                        }
                        .forEach { order ->
                            val subTotal = order.subtotal.toDouble()

                            order.cashOnHand = cash.toString()

                            if (order.isCheckAble) {
                                if (subTotal < cash) {
                                    order.isChecked = true
                                    order.status = AperdOrderStatus.READY_FOR_PROCESSING

                                    cash -= subTotal

                                    addCheck(order)
                                } else {
                                    order.isCheckAble = false
                                    order.status = AperdOrderStatus.UNPROCESSED
                                    order.unprocessedMsgList.add("No enough cash.")
                                }

                            }

                            orderBookingList[orderBookingList.indexOf(order)] = order
                        }
                }
            }
        }


#. *Function updateCheckedAllStatus()*
    *function* ``updateCheckedAllStatus()`` akan menceklis *checkbox toggle checked all*.


    .. code-block:: kotlin

        class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
            //other code...
            private fun updateCheckedAllStatus() {
                val checkedCount = checkedList.count()
                val checkAbleCount = orderBookingList.count { it.isCheckAble }

                isCheckedAll.value =  (checkAbleCount > 0)  && (checkedCount == checkAbleCount)
            }
        }


#. *Function updateSummaryTotalSection()*
    Terakhir, fungsi ini berguna untuk memperbaharui *summay section* pada layar.

    .. code-block:: kotlin

        class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
            //other code...
            private fun updateSummaryTotalSection() {
                totalItems.value = calcTotalItems()
                totalItemsProcessed.value = calcTotalProcessed()
                totalItemsUnprocessed.value = calcTotalUnprocessed()
                totalAmountProcessed.value = calcTotalAmountProcessed()
                totalFeeProcessed.value = calcTotalFeeProcessed()
                totalBcProcessed.value = calcTotalBcProcessed()
                grandTotal.value  = calcGrandTotal()
            }
        }


Kode terakhir berfungsi untuk menampilkan pesan *error* pada layar jika dalam block ``try {...}`` terdapat *error*,
dan menyimpan detail *error* pada *logger*.


.. code-block:: kotlin
        try {
            //other code...
        }
        catch (e: Exception) {
            Platform.runLater {
                Alerts.errors("Sorry, failed to process inquiry data, please try again later.")

                e.message?.let {
                    Logger.warning("${tagName}--processInquiryData", e.message!!)
                }
            }
        }



*Execute Bulk Order*
~~~~~~~

Pada proses ini akan membeli semua reksadana yang sudah diceklist oleh *user*. Akan berjalan ketika tombol *Execute*
ditekan dan akan menjalankan *function* ``executeBulkOrder()``.


.. code-block:: kotlin

    class BulkSubscription : Fragment("${AppProperties.appName} - Bulk Order Subscription Screen") {
        //other code...
        private fun executeBulkOrder() {
            val validator = Validator()
                .rule(orderBookingList.isNotEmpty(), "The table is empty, please upload the file again.")
                .rule(checkedList.isNotEmpty(), "Please choose one of the mutual funds.")
                .validate()

            if (!validator.isValid()) {
                Alerts.warning(validator.getErrorMessages().joinToString(separator  = "\n"))
                return
            }

            var bulkOrderSuccessful = false
            alert(
                Alert.AlertType.CONFIRMATION, "",
                "Are you sure you want to subscribe to the mutual funds?", ButtonType.YES,
                ButtonType.CANCEL, title = "Bulk Order Confirmation"
            ) { btnType ->
                if (btnType == ButtonType.YES) {
                    frgLoader.openModal(
                        stageStyle = StageStyle.TRANSPARENT,
                        modality = Modality.APPLICATION_MODAL,
                        escapeClosesWindow = false,
                        resizable = false,
                        owner = this@BulkSubscription.currentWindow
                    )
                    runAsync {
                        checkedList.forEachIndexed { idx, order ->
                            val user = userProfiles.find { it.custCode == order.saCode }

                            val subscribe = MutualFundOrder(
                                transDate = DateAndTime.now(),
                                transType = Constant.TRANS_TYPE_SUBS,
                                fundCode = order.fundCode,
                                sid = order.sid,
                                qtyAmount = order.amount,
                                qtyUnit = order.estUnit,
                                lastNav = order.lastNav,
                                feeNominal = order.trxFeeAmount,
                                feePersen = order.trxFee,
                                feeNominalDealer = order.dealerFeeAmount,
                                feePersenDealer = order.dealerFee,
                                redmPaymentAccSeqCode = "",
                                redmPaymentBicCode = "",
                                redmPaymentAccNo = "",
                                rdnAccNo = user?.rdncbAccNo,
                                rdnBankCode = Constant.bankInfo[user?.rdnBnCode]?.get("bank_code") ?: "",
                                rdnBankName = user?.rdnBnName,
                                cbAccNo = order.cbAccNo,
                                cbBankCode = order.cbSwiftCode,
                                cbBankName = order.cbBankName,
                                paymentDate = DateAndTime.now(),
                                transferType = order.transactionType,
                                transactionType = order.transferType,
                                bankCharge = order.bankCharge,
                                deviceId = Constant.DEVICE_ID_DESKTOP,
                                autoOrder = "",
                                dealerName = GlobalState.session.userId,
                                bulkOder = "y"
                            )

                            val result = runBlocking {
                                val webService = WebService()
                                webService.bulkSubscriptionOrder(subscribe)
                            }

                            if (result !== null) {
                                val msg = JSONObject(result)

                                try {
                                    ApiHelper.checkStatusResponse(msg)

                                    order.status = AperdOrderStatus.PROCESSED

                                    if (Config.DBG_WS_MARKETUPD) Logger.debug("bulkSubscriptionOrder--onMessage1", result)
                                } catch (e: Exception) {
                                    order.status = AperdOrderStatus.UNPROCESSED

                                    order.unprocessedMsgList.add("Failed to subscribe, please try again later")
                                } finally {
                                    order.isCheckAble = false
                                    order.isChecked = false
                                }

                            } else {
                                order.status = AperdOrderStatus.UNPROCESSED
                                order.unprocessedMsgList.add("Failed to subscribe, please try again later")
                            }

                            orderBookingList[orderBookingList.indexOf(order)] = order
                        }

                        bulkOrderSuccessful = true
                    } ui {
                        if (bulkOrderSuccessful) {
                            checkedList.clear()

                            validateCashOnHand()
                            updateCheckedAllStatus()
                            updateSummaryTotalSection()

                            frgLoader.close()

                            Alerts.information("The selected product has been processed.")
                        } else {
                            frgLoader.close()
                            Alerts.information("Bulk order processing failed. Please try again later.")
                        }
                    }
                }
            }
        }
    }


Pertama, melakukan validasi apakah data sudah ada yang dipilih atau belum

.. code-block:: kotlin

    val validator = Validator()
        .rule(orderBookingList.isNotEmpty(), "The table is empty, please upload the file again.")
        .rule(checkedList.isNotEmpty(), "Please choose one of the mutual funds.")
        .validate()

    if (!validator.isValid()) {
        Alerts.warning(validator.getErrorMessages().joinToString(separator  = "\n"))
        return
    }


Jika berhasil, akan mengubah nilai dari ``bulkOrderSuccessful`` menjadi *false* dan menampilkan pesan konfirmasi untuk
melakukan *bulk order*

.. code-block:: kotlin

    var bulkOrderSuccessful = false
    alert(
        Alert.AlertType.CONFIRMATION, "",
        "Are you sure you want to subscribe to the mutual funds?", ButtonType.YES,
        ButtonType.CANCEL, title = "Bulk Order Confirmation"
    ) { btnType ->
        if (btnType == ButtonType.YES) {
            //handle bulk order...
        }
    }


Setelah user menekan tombol *Yes* proses *bulk order* akan dilanjutkan, dan pertamakali akan menampilkan *loader indicator*.

.. code-block:: kotlin

    frgLoader.openModal(
        stageStyle = StageStyle.TRANSPARENT,
        modality = Modality.APPLICATION_MODAL,
        escapeClosesWindow = false,
        resizable = false,
        owner = this@BulkSubscription.currentWindow
    )


Selanjutnya, reksadana akan di *order* satu persatu seperti pada kode yang ditampilkan dibawah ini.

.. code-block:: kotlin

    runAsync {
        checkedList.forEachIndexed { idx, order ->
            val user = userProfiles.find { it.custCode == order.saCode }

            val subscribe = MutualFundOrder(
                transDate = DateAndTime.now(),
                transType = Constant.TRANS_TYPE_SUBS,
                fundCode = order.fundCode,
                sid = order.sid,
                qtyAmount = order.amount,
                qtyUnit = order.estUnit,
                lastNav = order.lastNav,
                feeNominal = order.trxFeeAmount,
                feePersen = order.trxFee,
                feeNominalDealer = order.dealerFeeAmount,
                feePersenDealer = order.dealerFee,
                redmPaymentAccSeqCode = "",
                redmPaymentBicCode = "",
                redmPaymentAccNo = "",
                rdnAccNo = user?.rdncbAccNo,
                rdnBankCode = Constant.bankInfo[user?.rdnBnCode]?.get("bank_code") ?: "",
                rdnBankName = user?.rdnBnName,
                cbAccNo = order.cbAccNo,
                cbBankCode = order.cbSwiftCode,
                cbBankName = order.cbBankName,
                paymentDate = DateAndTime.now(),
                transferType = order.transactionType,
                transactionType = order.transferType,
                bankCharge = order.bankCharge,
                deviceId = Constant.DEVICE_ID_DESKTOP,
                autoOrder = "",
                dealerName = GlobalState.session.userId,
                bulkOder = "y"
            )

            val result = runBlocking {
                val webService = WebService()
                webService.bulkSubscriptionOrder(subscribe)
            }
            //other code...
        }
        //other code...
    } ui {
        //other code...
    }


Setelah *request bulk order* ``webService.bulkSubscriptionOrder(subscribe)`` dilakukan, akan dicek apakah *response*
yang diterima berhasil atau tidak, jika tidak berhasil maka status menjadi *Unprocessed* dan menampilkan pesan *gagal*
ke layar.

.. code-block:: kotlin

    if (result !== null) {
        //other code...
    } else {
        order.status = AperdOrderStatus.UNPROCESSED
        order.unprocessedMsgList.add("Failed to subscribe, please try again later")
    }


Jika berhasil, status akan menjadi *Processed* yang menandakan *bulk order* berhasil dilakukan. Jika terdapat *error*
akan langsung mengubah status menjadi *Unprocessed* dan pesannya ditampilkan pada layar,
yang berada pada *block* ``catch (e: Exception) {...}``. Terakhir *checkbbox* akan *unchecked* dan *disabled*.

.. code-block:: kotlin

    try {
        ApiHelper.checkStatusResponse(msg)

        order.status = AperdOrderStatus.PROCESSED

        if (Config.DBG_WS_MARKETUPD) Logger.debug("bulkSubscriptionOrder--onMessage1", result)
    } catch (e: Exception) {
        order.status = AperdOrderStatus.UNPROCESSED

        order.unprocessedMsgList.add("Failed to subscribe, please try again later")
    } finally {
        order.isCheckAble = false
        order.isChecked = false
    }


Selanjutnya, memperbaharui data pada tabel, dan *variable* ``bulkOrderSuccessful`` menjadi *true*.

.. code-block:: kotlin

    private fun executeBulkOrder() {
        //other code...
        runAsync {
            checkedList.forEachIndexed { idx, order ->
                //other code...
                orderBookingList[orderBookingList.indexOf(order)] = order
            }
            bulkOrderSuccessful = true
        } ui {
            //other code...
        }
    }


Seteleh proses *request bulk order* selesai, akan menuju ke *block* ``ui {...}``.

.. code-block:: kotlin

    private fun executeBulkOrder() {
        runAsync {
            //other code...
        } ui {
            if (bulkOrderSuccessful) {
                checkedList.clear()

                validateCashOnHand()
                updateCheckedAllStatus()
                updateSummaryTotalSection()

                frgLoader.close()

                Alerts.information("The selected product has been processed.")
            } else {
                frgLoader.close()
                Alerts.information("Bulk order processing failed. Please try again later.")
            }
        }
    }


Pertama, akan melakukan pengecekan apakah proses *bulk order* berhasil dilakukan atau tidak. Jika gagal akan menampilkan
pesan error pada layar dan *loader indicator* dihilangkan.


.. code-block:: kotlin

    if (bulkOrderSuccessful) {
        //other code...
    } else {
        frgLoader.close()
        Alerts.information("Bulk order processing failed. Please try again later.")
    }



Kalau berhasil, data jumlah *checkbox* yang sudah diceklis akan dihapus ``checkedList.clear()``. *Cash on hand* akan
divalidasi lagi, *toggle checkbox all* akan di *uncheck* dan *summary section* diperbaharui. Detail dari setiap *functions*
bisa dilihat pada *section* *Process Inquiry Data*. Terakhir *loader indicator* akan dihilangkan dan menampilkan pesan
berhasil *bulk order*.


.. code-block:: kotlin

    if (bulkOrderSuccessful) {
        checkedList.clear()

        validateCashOnHand()
        updateCheckedAllStatus()
        updateSummaryTotalSection()

        frgLoader.close()

        Alerts.information("The selected product has been processed.")
    }  else {
        //other code...
    }



*Bulk Order History*
-------
*Bulk order history* akan menampilkan semua daftar *order* reksadana yang berahasil dilakukan pada proses *bulk order*.
Semua proses untuk menampilkan histori *bulk order* terdapat di *BulkOrderHistory Class* dan disimpan pada *file* ``bulkorderhistory.kt``.
Setelah menentukan tanggal, lanjut menekan tombol *submit* dan akan menajalankan *function* ``getOrderHistory()``
seperti kode dibawah ini.

.. code-block:: kotlin

    class BulkOrderHistory: Fragment("${AppProperties.appName} - Bulk Order Dealer History")  {
        //other code...
        private fun getOrderHistory() {
            val validator = Validator()
                .rule(frgDateRange.getStartDate() != null, "Start date is required.")
                .rule(frgDateRange.getEndDate() != null, "End date is required.")
                .rule(frgDateRange.isValidDate(), "Start date cannot be greater than end date.")
                .validate()
            if (!validator.isValid()) {
                Alerts.warning(validator.getErrorMessages().joinToString(separator = "\n"))
                return
            }

            val dealer = GlobalState.session.userId
            val startDate = frgDateRange.getStartDate()?.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"))
            val endDate =  frgDateRange.getEndDate()?.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"))

            frgLoader.openModal(
                stageStyle = StageStyle.TRANSPARENT,
                modality = Modality.APPLICATION_MODAL,
                escapeClosesWindow = false,
                resizable = false,
                owner = this@BulkOrderHistory.currentWindow
            )

            val task = runAsync {
                WebServiceData.historyTransactionByDealer(dealer, startDate.toString(), endDate.toString())
            }

            task.setOnSucceeded {
                val historyTransactions = task.value

                historyTransactions.sortByDescending { it.transactionDate }
                historyItems.setAll(historyTransactions)
                frgLoader.close()

                if (historyTransactions.isEmpty()) {
                    alertHistoryEmpty()
                }
            }

            task.setOnFailed {
                val exception = task.exception
                frgLoader.close()
                Alerts.errors(exception.message)
            }
        }
    }


Pertama, melakukan validasi agar format tanggal yang diberikan sesuai tidak ada yang bermasalah.

.. code-block:: kotlin

    val validator = Validator()
        .rule(frgDateRange.getStartDate() != null, "Start date is required.")
        .rule(frgDateRange.getEndDate() != null, "End date is required.")
        .rule(frgDateRange.isValidDate(), "Start date cannot be greater than end date.")
        .validate()
    if (!validator.isValid()) {
        Alerts.warning(validator.getErrorMessages().joinToString(separator = "\n"))
        return
    }


Selanjutnya, menentukan beberapa *variable* dan menampilkan *loader indicator* pada layar.

.. code-block:: kotlin

    val dealer = GlobalState.session.userId
    val startDate = frgDateRange.getStartDate()?.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"))
    val endDate =  frgDateRange.getEndDate()?.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"))

    frgLoader.openModal(
        stageStyle = StageStyle.TRANSPARENT,
        modality = Modality.APPLICATION_MODAL,
        escapeClosesWindow = false,
        resizable = false,
        owner = this@BulkOrderHistory.currentWindow
    )


*Request history transaction* untuk mendapatkan data *history* yang *bulk order* nya saja dari *server*.

.. code-block:: kotlin

    val task = runAsync {
        WebServiceData.historyTransactionByDealer(dealer, startDate.toString(), endDate.toString())
    }


Kalau gagal, akan menampilkan pesan *error* dan *loader indicator* dihilangkan dari layar.

.. code-block:: kotlin

    task.setOnFailed {
        val exception = task.exception
        frgLoader.close()
        Alerts.errors(exception.message)
    }


Jika berhasil, data akan diproses terlebih dahulu sebelum ditampilkan pada layar.

.. code-block:: kotlin

    task.setOnSucceeded {
        val historyTransactions = task.value

        historyTransactions.sortByDescending { it.transactionDate }
        historyItems.setAll(historyTransactions)
        frgLoader.close()

        if (historyTransactions.isEmpty()) {
            alertHistoryEmpty()
        }
    }


Data pertamakali akan di sorting secara *DESC* berdasarkan tanggal transaksi.

.. code-block:: kotlin

    historyTransactions.sortByDescending { it.transactionDate }


Setelah itu, data akan ditampilkan pada tabel, dan *loader indicator* akan dihilangkan pada layar.

.. code-block:: kotlin

    historyItems.setAll(historyTransactions)
    frgLoader.close()


Terakhir, akan menampilkan pesan pemberitahuan jika histori *bulk order* kosong.

.. code-block:: kotlin

    if (historyTransactions.isEmpty()) {
        alertHistoryEmpty()
    }


Berikut merupakan detail dari *function* ``alertHistoryEmpty()``.

.. code-block:: kotlin

    class BulkOrderHistory: Fragment("${AppProperties.appName} - Bulk Order Dealer History")  {
        //other code...
        private fun alertHistoryEmpty() {
            val startDate = frgDateRange.getStartDate()?.format(DateTimeFormatter.ofPattern("dd/MM/yyyy"))
            val endDate = frgDateRange.getEndDate()?.format(DateTimeFormatter.ofPattern("dd/MM/yyyy"))

            Alerts.warning("Sorry, there is no historical data from $startDate to $endDate.")
        }
    }


.. autosummary::
   :toctree: generated