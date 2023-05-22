# inappbilling2




class MainActivity : AppCompatActivity() {

    private lateinit var billingClient:BillingClient
    private var isConnected=false

    private lateinit var productDetailss:MutableList<ProductDetails>


    private lateinit var binding:ActivityMainBinding
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding=ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)


        val pUL= PurchasesUpdatedListener { billingResult, purchases ->

            //launch billing denildikten sonra...
            if (billingResult.responseCode==BillingResponseCode.OK && purchases!=null){

                for (i in purchases){

                    //handlePurchaseNonConsumableProduct(i)
                    handlePurchaseConsumableProduct(i)

                }

                println("pUL tetiklendi!")
            }


        }


        billingClient=BillingClient.newBuilder(this).enablePendingPurchases()
            .setListener(pUL).build()

        startBillingClientConnection()

        binding.show.setOnClickListener {

            showProducts()
        }

        binding.button.setOnClickListener {

            buy30Elmas()

        }



    }

    //initialize
    private fun startBillingClientConnection(){

        billingClient.startConnection(object : BillingClientStateListener{
            override fun onBillingServiceDisconnected() {

                //try reconnect
                isConnected=false
                println("disconnected!!")

            }

            override fun onBillingSetupFinished(billingResult: BillingResult) {

                if (billingResult.responseCode==BillingResponseCode.OK){

                    //query purchase here
                    isConnected=true
                    println("connected!!")


                }

            }
        })

    }


    private fun showProducts(){

        val productList= listOf(
            QueryProductDetailsParams.Product.newBuilder().setProductId("elmas_30")
                .setProductType(ProductType.INAPP)
                .build()
        )

        val queryProductDetailsParams=QueryProductDetailsParams.newBuilder()
            .setProductList(productList)
            .build()

        billingClient.queryProductDetailsAsync(queryProductDetailsParams){ billingResult,productDetails->

            if (billingResult.responseCode==BillingResponseCode.OK){

                println(productDetails)
                productDetailss=productDetails
                binding.textView.text=productDetails[0].title
                binding.button.text=productDetails[0].oneTimePurchaseOfferDetails?.formattedPrice
            }

        }


    }


    private fun buy30Elmas(){

        val productDetailsParamsList=
            listOf(BillingFlowParams.ProductDetailsParams.newBuilder()

                .setProductDetails(productDetailss[0]).build()
            )

        val billingFlowParams=
            BillingFlowParams.newBuilder()
                .setProductDetailsParamsList(productDetailsParamsList)
                .build()

        val billingResult =
            billingClient.launchBillingFlow(this@MainActivity,billingFlowParams)

    }

    //tüketilmeyen ve yalnızca 1 kere alınabilen ürünlerin doğrulanması
    private fun handlePurchaseNonConsumableProduct(purchase: Purchase){


        if (purchase.purchaseState==Purchase.PurchaseState.PURCHASED){

            if (!purchase.isAcknowledged){

                val acknowledgePurchaseParams=
                    AcknowledgePurchaseParams.newBuilder()
                        .setPurchaseToken(purchase.purchaseToken)

                CoroutineScope(Dispatchers.IO).launch {

                    val acknowledgePurchaseResult=
                        billingClient.acknowledgePurchase(acknowledgePurchaseParams.build()) {

                            println("satın alma doğrulandı!")
                            println(it)

                        }


                }


            }


        }

    }


    //tüketilen ve birden fazla kez alınabilen ürünlerin doğrulanması
    private fun handlePurchaseConsumableProduct(purchase: Purchase){

        val consumeParams=
            ConsumeParams.newBuilder()
                .setPurchaseToken(purchase.purchaseToken)
                .build()

        val consumeResult= CoroutineScope(Dispatchers.IO).launch {

            billingClient.consumePurchase(consumeParams)

        }

        verification(purchase)

    }

    private fun verification(purchase: Purchase){

        if(purchase.purchaseState==Purchase.PurchaseState.PURCHASED){

            //security verify işlemi
            if(!verifyValidSignature(purchase.originalJson,purchase.signature)){
                //doğrulama başarısız..
                println("doğrulama başarısız")
                return
            }


            println("doğrulama başarılı")

        }

    }

    private fun verifyValidSignature(signedData:String, signature:String):Boolean{

        return try{
            val base64Key="Your app license code get in Google play account monetizing section"
            val security=Security()

            security.verifyPurchase(base64Key,signedData,signature)
        } catch(e: Exception){
            false
        }


    }

    override fun onDestroy() {
        super.onDestroy()

        billingClient.endConnection()
    }
}
