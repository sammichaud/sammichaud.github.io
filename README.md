# Projet Porte

## Introduction
Ce projet a pour objectif de permettre la programmation de tags NFC depuis la page web du système de gestion d'accès au travers d'une application android.

Pour cela l'application s'ouvre avec un lecteur de QR code. Une fois ce QR analysé, l'application va charger la page Web correspondante dans une [WebView][1]. Celle-ci offre une API JavaScript à la page Web afin de permettre d'utiliser les fonctions NFC natives à Android.

## QR code
Pour la partie QR code l'application [Barcode Scanner][2] a été intégrer dans l'application. Pour cela nous avons suivi ce tuto: https://github.com/zxing/zxing/wiki/Scanning-Via-Intent.
```Java
IntentIntegrator integrator = new IntentIntegrator(activity);
integrator.initiateScan();
```
Dans cette partie on crée l'objet scan puis on lance le scan juste en dessous.
```Java
public void onActivityResult(int requestCode, int resultCode, Intent intent) {
        IntentResult scanResult = IntentIntegrator.parseActivityResult(requestCode, resultCode, intent);
        if (scanResult != null) {
            // Quand le scan a détecter un QR
            intent = new Intent(MainActivity.this, MainWebView.class);
            intent.putExtra("URL_ID", scanResult.getContents().toString());
            startActivity(intent);
        }
        // else continue with any other code you need in the method
    }
```
onActivityResult se lance lorsque le scan a trouvé un QR code à lire. Il fait un contrôle du résulta et continue que s'il n'est pas vide. Ensuite on récupère le résultat du scan qui est une URL puis on le transmet avec putExtra à la prochaine page qui est le WebView qui va afficher cette page.

## WebView
On crée la WebView et on la relit (bind) avec la classe [WebAppInterface](#partie-java-webappinterface-) pour relier la partie de code JavaScript qui se trouve dans la page web et le code Java qui se trouve elle dans l'application.
```Java
webAppInterface = new WebAppInterface(this);
WebView myWebView = (WebView) findViewById(R.id.webview);
myWebView.addJavascriptInterface(webAppInterface, "Android");
WebSettings webSettings = myWebView.getSettings();
webSettings.setJavaScriptEnabled(true);
```
L'intégration de la page web à la WebView se fait avec la fonction loadUrl.
```Java
final Activity activity = this;
        myWebView.setWebChromeClient(new WebChromeClient() {
            public void onProgressChanged(WebView view, int progress) {
                activity.setProgress(progress * 1000);
            }
        });
        myWebView.setWebViewClient(new WebViewClient() {
            public void onReceivedError(WebView view, int errorCode, String description, String failingUrl) {
                Toast.makeText(activity, "Oh no! " + description, Toast.LENGTH_SHORT).show();
            }
        });
        myWebView.loadUrl(URL);
```

### Interface JavaScript
Cette section se sépare en deux parties : la partie JavaScript qui appelle les fonctions depuis le navigateur et la partie Java qui traite les données transmises.

#### Partie JavaScript
L'API JavaScript développée propose 3 fonctions :
1. `programTag(timestamp, tags, key)` écrit sur le tag la date d'échéance(timestamp), écrite la liste de toute les portes, hashe la mémoire, la cryte avec la clé(key) puis la verouille le tag.
2. `liberer()` déverouille le tag.
3. `EcrireURL(URL)` déverouille le tag puis écrit sur un tag l'URL passé en paramètre.

##### Exemple
Ici sur le click d'un bouton on appelle la fonction JavaScript WriteURL() qui elle appelle la fonction Java EcrireURL() en lui passant en paramètre une URL
```javascript
function WriteURL(){
  var URL = "https://www.rts.ch/";
  Android.EcrireURL(URL);
}

<input type="button" value="Ecrire URL" onClick="WriteURL()" />
```
#### Partie Java (WebAppInterface)
La classe WebAppInterface permet de fournir [l'interface JavaScript][3] à la WebView. Il faut passer à la WebView une instance de cette classe et toutes les fonctions `@JavascriptInterface` seront disponibles.

##### Exemple
On se trouve dans la fonction Java EcrireURL qui est appeller dans le JavaScript. On récupère l'URL et on le traite pour l'écrire sur le Tag NFC on verra comment juste après.
```Java
public class WebAppInterface {
    @JavascriptInterface
    public void EcrireURL(String url){
        handleType = HandleType.URL;
        this.url = url;

        NfcAdapter mNfcAdapter = NfcAdapter.getDefaultAdapter(context);
        if (!mNfcAdapter.isEnabled()) {
            Toast.makeText(context, "NFC is disabled", Toast.LENGTH_SHORT).show();
        }else {
            newFragment.setParameters((MainWebView) context);
            newFragment.show(((Activity)context).getFragmentManager(), "missiles");
            ((MainWebView) context).setupForegroundDispatch((Activity) context, mNfcAdapter);
        }
    }
}
```
## NFC
L'application utilise le [NFC natif][4] d'Android. Il y a trois fonctions différentes :

1. Fontion pour écrire sur le tag le timestamp et la liste des portes
2. Fonction pour déverouiller le tag
3. Fonction pour écrit une URL dessus le tag

Ces fonctions lancent, grâce à la fonction `setupForegroundDispatch`, le scan d'un tag NFC.

```Java
public static void setupForegroundDispatch(final Activity activity, NfcAdapter adapter) {
       final Intent intent = new Intent(activity.getApplicationContext(), activity.getClass());
       intent.setFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);

       final PendingIntent pendingIntent = PendingIntent.getActivity(activity.getApplicationContext(), 0, intent, 0);
       String[][] techList = new String[][]{};

       adapter.enableForegroundDispatch(activity, pendingIntent, null, techList);
   }
```
Une fois qu'un tag a été trouvé la fonction suivante s'exécute. Selon la fonction JavaScript  appelée, le tag sera traité de façon différente :

```java
@Override
    protected void onNewIntent(Intent intent) {
        switch(webAppInterface.handleType) {
            case Ecrire:
                nfcWriteAndLock(intent);
                break;
            case Unlock:
                nfcUnlock(intent);
                break;
            case WriteURL:
                nfcWriteUrl(intent);
                break;
        }
    }
```

Il y a trois possibilités:
1. [Ecrire](#fonction-nfcwriteandlock) (nfcWriteAndLock)
2. [Unlock](#fonction-nfcunlock)  (nfcUnlock)
3. [WriteURL](#fonction-nfcwriteurl) (nfcWriteUrl)

Toutes ces fonctions se basent sur la classe NfcUtil. Des informations sur cette classe sont disponibles dans la javadoc.

#### Fonction nfcWriteAndLock
Dans ce cas la fontion va écrire sur le tag le timestamp et la liste des portes. Pour ce faire elle va récuperer le tag, écrire la date d'échéance(timestamp) dessus, écrire la liste des portes, recupérer toute la mémoire du tag et la stockée, hasher la mémoire stocké, crypter le hashage de la mémoire puis la réécrire sur le tag pour finalement bloquer le tag avec un mot de passe.
```Java
 NfcUtil.writeTimestamp(tag, webAppInterface.timestamp);
 NfcUtil.writeDoor(tag);
 byte[] memory = NfcUtil.readTag(tag);
 byte[] hash = NfcUtil.hashMemory(memory);
 byte[] sign = NfcUtil.signHash(hash, key);
 NfcUtil.writeSign(tag, sign);
 NfcUtil.lock(tag);
 webAppInterface.newFragment.dismiss();
 Toast.makeText(this, "Ecriture terminée", Toast.LENGTH_SHORT).show();
```
#### Fonction nfcUnlock
Ici la fonction va déverrouiller le tag.
```Java
NfcUtil.unlock(tag);
webAppInterface.newFragment.dismiss();
Toast.makeText(this, "Déverrouillage terminé", Toast.LENGTH_SHORT).show();
```
#### Fonction nfcWriteUrl
Cette fontion va dévérouiller le tag, puis écire une URL dessus.
```Java
NfcUtil.unlock(tag);
NfcUtil.writeURL(tag, webAppInterface.url);
webAppInterface.newFragment.dismiss();
Toast.makeText(this, "Ecriture d'URL terminé", Toast.LENGTH_SHORT).show();
```

[1]: https://developer.android.com/reference/android/webkit/WebView.html
[2]: https://play.google.com/store/apps/details?id=com.google.zxing.client.android
[3]: https://developer.android.com/guide/webapps/webview.html
[4]: https://developer.android.com/guide/topics/connectivity/nfc/nfc.html
