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

- *Function AddProductToTable()*
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


- *Function updateSummaryTotalSection()*
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


- *Function resetInputs()*
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


Pertama, menentukan *local variable* untuk digunakan nanti pada proses selanjutnya.

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

- *Function addPortofolioToTable()*
    *Function* ini berguna untuk menambahkan data reksadana yang mau dijual pada tabel. Data yang berada pada tabel
    disimpan di *variable* ``fundOrders``. Proses penambahan data ini tidak langsung disimpan begitu saja, tetapi
    melewati pengecekan terlebih dahulu.
    Mengecek apakah reksadana sudah ada yang disimpan atau belum, seperti pada *code* ``if (dataToUpdate != null) {}``.
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


- *Function updateSummaryTotalSection()*
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


- *Function resetInputs()*
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

Menentukan besaran nominal *amount* atau *unit* yang mau dilakukan pemindahan dengan menenkan tombol *Add*, lalu
akan menjalankan *code* yang berada pada *function* ``addSwitchingIn()``

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



*Bulk Order History*
-------



.. autosummary::
   :toctree: generated