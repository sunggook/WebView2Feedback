Server Certificate API
===
# Background

The WebView2 team has been asked for an API to intercept when WebView2 cannot verify server's digital certificate while loading a web page.
This API provides an option to trust the server's TLS certificate at the application level and render the page without prompting the user about the TLS error or can cancel the request.

# Description

We propose adding `ReceivingServerCertificateError` API that allows you to verify TLS certificate,  and either continue the request
to load the resource or cancel the request.

When the event is raised, WebView2 will pass a `CoreWebView2ReceivingServerCertificateErrorEventArgs` , which lets you view the TLS certificate request Uri, error information, inspect the certificate metadata and several options for responding to the TLS request.

* You can perform your own verification of the certificate and allow the navigation to proceed if you trust it.
* You can choose to cancel the request.
* You can choose to display the default TLS interstitial page to let user respond to the request.

We also propose adding a `ClearServerCertificateErrorOverrideCache` API that clears certificate decisions that were added in response to proceeding with TLS certificate errors.

# Examples

## Win32 C++
``` cpp
// This example bypasses the default TLS interstitial page using the ReceivingServerCertificateError
// event handler and continues with the navigation to a server with a TLS certificate that is signed by
// an authority that WebView2 doesn't trust. Otherwise, cancel the request.
void SettingsComponent::EnableCustomServerCertificateError()
{
  if (m_webview)
  {
    if (m_ServerCertificateErrorToken.value == 0)
    {
         CHECK_FAILURE(m_webview->add_ReceivingServerCertificateError(
             Callback<ICoreWebView2ReceivingServerCertificateErrorEventHandler>(
             [this](
               ICoreWebView2* sender,
               ICoreWebView2ReceivingServerCertificateErrorEventArgs* args) {
               COREWEBVIEW2_WEB_ERROR_STATUS errorStatus;
               CHECK_FAILURE(args->get_ErrorStatus(&errorStatus));

               wil::com_ptr<ICoreWebView2Certificate> certificate = nullptr;
               CHECK_FAILURE(args->get_ServerCertificate(&certificate));

               // Continues with the navigation to a server with a TLS certificate if
               // the certificate is invalid and signed by badssl authority.
               if (errorStatus == COREWEBVIEW2_WEB_ERROR_STATUS_CERTIFICATE_IS_INVALID &&
                        (certificateAuthority.get() == L"*.badssl.com" ||
                        certificateAuthority.get() == L"BadSSL Untrusted Root Certificate Authority"))
               {
                 CHECK_FAILURE(args->put_Handled(TRUE));
               }
               else
               {
                 // Cancel the request.
                 CHECK_FAILURE(args->put_Cancel(TRUE));
               }
               return S_OK;
             })
             .Get(),
         &m_ServerCertificateErrorToken));
    }
    else
    {
      CHECK_FAILURE(m_webview->remove_ReceivingServerCertificateError(
        m_ServerCertificateErrorToken));
      m_ServerCertificateErrorToken.value = 0;
    }
  }
  else
  {
    FeatureNotAvailable();
  }
}

// This example clears the TLS decision in response to proceeding with TLS certificate errors.
if (m_webview)
{.
  CHECK_FAILURE(m_webView->ClearServerCertificateErrorOverrideCache(
        Callback<
            ICoreWebView2ClearServerCertificateErrorOverrideCacheCompletedHandler>(
            [](HRESULT error, PCWSTR result) -> HRESULT {
                CHECK_FAILURE(error);
                if (error != S_OK)
                {
                  ShowFailure( error,  L"Clear server certificate error override cache failed");
                }
                MessageBox(nullptr, L"Cleared", L"Clear server certificate error override cache", MB_OK);
                return S_OK;
            })
            .Get()));
}
```
## . NET/ WinRT
```c#
// This example bypasses the default TLS interstitial page using the ReceivingServerCertificateError
// event handler and continues with the navigation to a server with a TLS certificate that is signed by
// an authority that WebView2 doesn't trust. Otherwise, cancel the request.
private bool _isServerCertificateError = false;
void EnableCustomServerCertificateError()
{
  // Safeguarding the handler when unsupported runtime is used.
  try
  {
    if (!_isServerCertificateError)
    {
      webView.CoreWebView2.ReceivingServerCertificateError += WebView_ReceivingServerCertificateError;
    }
    else
    {
      webView.CoreWebView2.ReceivingServerCertificateError -= WebView_ReceivingServerCertificateError;
    }
    _isServerCertificateError = !_isServerCertificateError;

    MessageBox.Show(this,
      _isServerCertificateError ? "Custom server certificate error has been enabled" : "Custom server certificate error has been disabled",
      "Custom server certificate error");
  }
  catch (NotImplementedException exception)
  {
    MessageBox.Show(this, "Custom server certificate error Failed: " + exception.Message, "Custom server certificate error");
  }
}

void WebView_ReceivingServerCertificateError(object sender, CoreWebView2ReceivingServerCertificateErrorEventArgs e)
{
  CoreWebView2Certificate certificate = e.ServerCertificate;

  // Continues with the navigation to a server with a TLS certificate if
  // the certificate is invalid and signed by badssl authority.
  if (e.ErrorStatus == CoreWebView2WebErrorStatus.CertificateIsInvalid &&
      (certificate.Issuer == "*.badssl.com" ||
      certificate.Issuer == "BadSSL Untrusted Root Certificate Authority"))
  {
    // Continue the request with the TLS certificate.
    e.Handled = true;
  }
  else
  {
    // Cancel the request.
    e.Cancel = true;
  }
}

// This example clears the TLS decision in response to proceeding with TLS certificate errors.
void ClearServerCertificateErrorOverrideCache()
{
  await webView.CoreWebView2.ClearServerCertificateErrorOverrideCacheAsync();
  MessageBox.Show(this, "Cleared", "Clear server certificate error override cache");
}
```

# API Details

## Win32 C++
``` cpp
[uuid(4B7FF0D2-8203-48B0-ACBF-ED9CFF82567A), object, pointer_default(unique)]
interface ICoreWebView2_11 : ICoreWebView2_10 {
  /// Add an event handler for the ServerCertificateError event.
  /// The ServerCertificateError event is raised when the WebView2
  /// cannot verify server's digital certificate while loading a web page.
  ///
  /// With this event you have several options for responding to server
  /// certificate error requests:
  ///
  /// Scenario                                                   | Handled | Cancel
  /// ---------------------------------------------------------- | ------- | ------
  /// Ignore the warning and continue the request                | True    | False
  /// Cancel the request                                         | n/a     | True
  /// Display default TLS interstitial page                      | False   | False
  ///
  /// If you don't handle the event, WebView2 will show the default TLS interstitial page to user.
  ///
  /// WebView2 caches the response when `Handled` is set `TRUE` for the host and certificate in the session
  /// and ServerCertificateError event won't be raised again.
  ///
  /// To raise the event again you must clear the cache using `ClearServerCertificateErrorOverrideCache`.
  ///
  /// \snippet SettingsComponent.cpp ServerCertificateError1
  HRESULT add_ReceivingServerCertificateError(
      [in] ICoreWebView2ReceivingServerCertificateErrorEventHandler* eventHandler,
      [out] EventRegistrationToken * token);
  /// Remove an event handler previously added with add_ReceivingServerCertificateError.
  HRESULT remove_ReceivingServerCertificateError([in] EventRegistrationToken token);

  /// Clears the TLS decision in response to proceeding with TLS certificate errors.
  HRESULT ClearServerCertificateErrorOverrideCache(
      [in] ICoreWebView2ClearServerCertificateErrorOverrideCacheCompletedHandler*
      handler);
}

/// Receives the result of the `ClearServerCertificateErrorOverrideCache` method.
[uuid(2F7B173D-3CE1-4945-BDE6-94F4C57B7209), object, pointer_default(unique)]
interface ICoreWebView2ClearServerCertificateErrorOverrideCacheCompletedHandler : IUnknown {
  /// Provides the result of the corresponding asynchronous method.
  HRESULT Invoke([in] HRESULT errorCode, BOOL isSuccessful);
}

/// An event handler for the `ReceivingServerCertificateError` event.
[uuid(AAC28793-11FC-4EE5-A8D4-25A0279B1551), object, pointer_default(unique)]
interface ICoreWebView2ReceivingServerCertificateErrorEventHandler : IUnknown {
  /// Provides the event args for the corresponding event.
  HRESULT Invoke([in] ICoreWebView2* sender,
                 [in] ICoreWebView2ReceivingServerCertificateErrorEventArgs*
                     args);
}

/// Event args for the `ReceivingServerCertificateError` event.
[uuid(24EADEE7-31F9-447F-9FE7-7C13DC738C32), object, pointer_default(unique)]
interface ICoreWebView2ReceivingServerCertificateErrorEventArgs : IUnknown {
  /// The TLS error code for the invalid certificate.
  [propget] HRESULT ErrorStatus([out, retval] COREWEBVIEW2_WEB_ERROR_STATUS* value);

  /// URI associated with the request for the invalid certificate.
  [propget] HRESULT RequestUri([out, retval] LPWSTR* value);

  /// Returns the server certificate.
  [propget] HRESULT ServerCertificate([out, retval] ICoreWebView2Certificate** value);

  /// You may set this flag to cancel the request. The request is canceled regardless
  /// of the `Handled` property. By default the value is `FALSE`.
  [propget] HRESULT Cancel([out, retval] BOOL* value);

  /// Sets the `Cancel` property.
  [propput] HRESULT Cancel([in] BOOL value);

  /// You may set this flag to `TRUE` to continue the request with the TLS certificate.
  /// By default the value of `Handled` and `Cancel` are `FALSE` and display default TLS
  /// interstitial page to allow the user to take the decision.
  [propget] HRESULT Handled([out, retval] BOOL* value);

  /// Sets the `Handled` property.
  [propput] HRESULT Handled([in] BOOL value);

  /// Returns an `ICoreWebView2Deferral` object. Use this operation to
  /// complete the event at a later time.
  HRESULT GetDeferral([out, retval] ICoreWebView2Deferral** deferral);
}
```

```c# (but really MIDL3)
namespace Microsoft.Web.WebView2.Core
{
    runtimeclass CoreWebView2ReceivingServerCertificateErrorEventArgs
    {
        CoreWebView2WebErrorStatus ErrorStatus { get; };
        String RequestUri { get; };
        CoreWebView2Certificate ServerCertificate { get; };
        Boolean Cancel { get; set; };
        Boolean Handled { get; set; };
        Windows.Foundation.Deferral GetDeferral();
    }

    runtimeclass CoreWebView2
    {
        // ...
        [interface_name("Microsoft.Web.WebView2.Core.ICoreWebView2_11")]
        {
          event Windows.Foundation.TypedEventHandler<CoreWebView2, CoreWebView2ReceivingServerCertificateErrorEventArgs> ReceivingServerCertificateError;
          void ClearServerCertificateErrorOverrideCacheAsync();
        }
    }
}
```