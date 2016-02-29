# android4.0以上webview加载SSl证书兼容性问题

来源:[CSDN](http://blog.csdn.net/csh159/article/details/24358343)

```
下面参考了其他博客获取证书。  
public class AppConfig  {  
  
    private static WebView mWebView;  
  
    public static X509Certificate[] mX509Certificates;  
  
    public static PrivateKey mPrivateKey;  
  
    public static String CERTFILE_PASSWORD;  
  
    private static void clearParent()  {  
        if (mWebView != null)  {  
            ViewGroup p = (ViewGroup) mWebView.getParent();  
            if (p != null)  {  
                p.removeAllViewsInLayout();  
            }  
        }  
    }  
  
    public static void initPrivateKeyAndX509Certificate(Context c)  
            throws Exception  {  
        if (mPrivateKey != null && mX509Certificates != null)  {  
            return;  
        }  
        KeyStore keyStore = KeyStore.getInstance("PKCS12", "BC");  
        keyStore.load(c.getResources().openRawResource(R.raw.dianshang),  
                CERTFILE_PASSWORD.toCharArray());  
        Enumeration<?> localEnumeration;  
        localEnumeration = keyStore.aliases();  
        while (localEnumeration.hasMoreElements())  {
            String str3 = (String) localEnumeration.nextElement();  
            mPrivateKey =  
                    (PrivateKey) keyStore.getKey(str3,  
                            CERTFILE_PASSWORD.toCharArray());  
            if (mPrivateKey == null)  {  
                continue;  
            }  
            else  {  
                Certificate[] arrayOfCertificate =  
                        keyStore.getCertificateChain(str3);  
                mX509Certificates =  
                        new X509Certificate[arrayOfCertificate.length];  
                for (int j = 0; j < mX509Certificates.length; j++)  {  
                    mX509Certificates[j] =  
                            ((X509Certificate) arrayOfCertificate[j]);  
                }  
            }  
        }  
    }  
  
    public static void initPrivateKeyAndX509Certificate(Context c,  
            InputStream in) throws Exception  {  
        if (mPrivateKey != null && mX509Certificates != null)  {  
            return;  
        }  
        KeyStore keyStore = KeyStore.getInstance("PKCS12", "BC");  
        keyStore.load(in, CERTFILE_PASSWORD.toCharArray());  
        Enumeration<?> localEnumeration;  
        localEnumeration = keyStore.aliases();  
        while (localEnumeration.hasMoreElements())  {  
            String str3 = (String) localEnumeration.nextElement();  
            mPrivateKey =  
                    (PrivateKey) keyStore.getKey(str3,  
                            CERTFILE_PASSWORD.toCharArray());  
            if (mPrivateKey == null)  {  
                continue;  
            }  
            else  {  
                Certificate[] arrayOfCertificate =  
                        keyStore.getCertificateChain(str3);  
                mX509Certificates =  
                        new X509Certificate[arrayOfCertificate.length];  
                for (int j = 0; j < mX509Certificates.length; j++)  {  
                    mX509Certificates[j] =  
                            ((X509Certificate) arrayOfCertificate[j]);  
                }  
            }  
        }  
    }}  

```

下面是复写WebViewClient或者WebViewClientClassicExt（）具体兼容性问题可以参考下面类的注释：

```
/** 
 * 4.0-4.1可以继承WebViewClient类实现下面的方法（4.1以上不兼容，执行不了类里面的加载证书的方法） 
 * 但是在4.1以上上面的方法不可用，必须继承WebViewClientClassicExt方可实现证书加载(4.1以上部分方法的调用放到这个类里面了) 
 * @author dennis 
 * 
 */  
public class AppWebViewClientEx extends WebViewClientClassicExt  {  
  
    public AppWebViewClientEx(Context c)  {  
    }  
  
    public void onReceivedClientCertRequest(WebView view,  
            ClientCertRequestHandler handler, String host_and_port)  {  
        if ((null != AppConfig.mPrivateKey)  
                && ((null != AppConfig.mX509Certificates) && (AppConfig.mX509Certificates.length != 0)))  {  
            handler.proceed(AppConfig.mPrivateKey, AppConfig.mX509Certificates);  
        }  
        else  {  
            handler.cancel();  
        }  
    }  
  
    @Override  
    public void onReceivedSslError(final WebView view, SslErrorHandler handler,  
            SslError error)  {  
        handler.proceed();  
    }  
}  
```

使用隐藏api：

这个问题弄了好久终于搞定了4.0以上webview加载ssl证书不兼容问题....纠结死了....