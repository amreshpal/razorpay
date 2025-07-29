# razorpay

import 'dart:convert';

import 'package:crypto/crypto.dart';
import 'package:flutter/material.dart';
import 'package:razorpay_flutter/razorpay_flutter.dart';
import 'package:http/http.dart' as http;

class RazorPayScreen extends StatefulWidget {
  const RazorPayScreen({super.key});

  @override
  State<RazorPayScreen> createState() => _RazorPayScreenState();
}

class _RazorPayScreenState extends State<RazorPayScreen> {
  final _razorpay = Razorpay();
  bool verifySignature({
    required String orderId,
    required String paymentId,
    required String signature,
    required String keySecret,
  }) {
    final key = utf8.encode(keySecret);
    final bytes = utf8.encode('$orderId|$paymentId'); // ✅ As per Razorpay docs

    final hmacSha256 = Hmac(sha256, key);
    final digest = hmacSha256.convert(bytes);
    final generatedSignature = digest.toString();

    print("Generated Signature: $generatedSignature");
    print("Received Signature: $signature");

    return generatedSignature == signature;
  }

  String orderId = "";
  @override
  void initState() {
    _razorpay.on(Razorpay.EVENT_PAYMENT_SUCCESS, _handlePaymentSuccess);
    _razorpay.on(Razorpay.EVENT_PAYMENT_ERROR, _handlePaymentError);
    _razorpay.on(Razorpay.EVENT_EXTERNAL_WALLET, _handleExternalWallet);
    super.initState();
  }

  void _handlePaymentSuccess(PaymentSuccessResponse response) {
    final isValid = verifySignature(
      orderId: response.orderId!,
      paymentId: response.paymentId!,
      signature: response.signature!,
      keySecret: '1Erv9zh4aB22mXrkYzGnhQHN', // 🚫 NEVER hardcode in production
    );

    if (isValid) {
      print("✅ Payment Verified");
    } else {
      print("❌ Payment Signature Mismatch");
    }
    print("response data  ${response.data}");
    print(response.signature);
    print(response.orderId);
    print(response.paymentId);
    // Do something when payment succeeds
  }

  void _handlePaymentError(PaymentFailureResponse response) {
    print("response or failed data ${response.error}");
    // Do something when payment fails
  }

  void _handleExternalWallet(ExternalWalletResponse response) {}

  @override
  void dispose() {
    _razorpay.clear(); // Removes all listeners
    super.dispose();
  }

  Future<String?> createRazorpayOrder() async {
    const keyId = 'rzp_test_TXuWjyJ7mDCpGd';
    const keySecret = '1Erv9zh4aB22mXrkYzGnhQHN';

    String basicAuth =
        'Basic ' + base64Encode(utf8.encode('$keyId:$keySecret'));

    final response = await http.post(
      Uri.parse('https://api.razorpay.com/v1/orders'),
      headers: {
        'Content-Type': 'application/json',
        'Authorization': basicAuth,
      },
      body: jsonEncode({
        "amount": 5000,
        "currency": "INR",
        "receipt": "qwsaq1",
        "partial_payment": true,
        "first_payment_min_amount": 230,
        "notes": {"key1": "value3", "key2": "value2"}
      }),
    );

    print("Status Code: ${response.statusCode}");
    print("Response: ${response.body}");

    if (response.statusCode == 200) {
      final data = jsonDecode(response.body);
      return data['id']; // ✅ Razorpay order_id
    } else {
      print("Failed to create order");
      return null;
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
        appBar: AppBar(
          title: Text("RazorPay Screen"),
        ),
        body: Center(
            child: ElevatedButton(
          onPressed: () async {
            final orderId = await createRazorpayOrder();

            if (orderId != null) {
              var options = {
                'key': 'rzp_test_TXuWjyJ7mDCpGd',
                'amount': 5000,
                'currency': 'INR',
                'name': 'Acme Corp.',
                'order_id': orderId, // ✅ now it's not null
                'description': 'Fine T-Shirt',
                'timeout': 60,
                'prefill': {
                  'contact': '+917081461044',
                  'email': 'gaurav.kumar@example.com'
                },
              };
              _razorpay.open(options);
            }
          },
          child: Text("Pay With RazorPay"),
        )));
  }
}
