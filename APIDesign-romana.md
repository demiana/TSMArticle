## Designul de API - studiu de caz ##

În zilele noastre, pentru orice produs sau serviciu este din ce în ce mai greu sa devină de succes, de aici rezultând și nevoia expunerii unui *API*(acronim pentru *Application Programming Interface*). Un *API* defineste în mod clar cum interactionează un sistem *software*  sau o componentă cu multitudinea de sisteme existente. Printre avantajele unui *API* se numară lucruri ca și scalabilitate, flexibilitate si inovație. Marile companii printre care se numară Microsoft, Google, Facebook sau Twitter nu s-ar afla în momentul de față la acest nivel dacă nu ar fi beneficiat de pe urma expunerii propriilor *API*-uri.

### De ce este așa grea definirea unui API?###

Designul de *API* este unul din lucrurile care pot contribui semnificativ la creșterea valorii unui serviciu. Cu toate acestea definirea unei noi interfețe reușite este o muncă dificilă. Orice dezvoltator *software* știe cât de ușor poate codul unui proiect să evolueze într-un cod *"spaghetti"*. API*-urile nu sunt nici ele ferite de această degradare. Un *API* bine definit trebuie să reflecte obiectivele *businessului* din care face parte dar tot o dată să țină cont și de punctele forte și limitările organizației cu privire la buget, competențe personale și infrastructura tehnică. Cheia întregului process este să fie foarte clar ce problema rezolvă această nouă interfața. Mai sunt și alte aspecte care trebuie luate în calcul pentru definirea unui *API* reușit. Un *API*
trebuie să fie:
 
- Să fie ușor de extins și de integrat in aplicația clientului
- Greu de folosit într-un mod incorect
- Codul clientului care îl integrează nu este greu de citit și de întreținut
- Capabil să satisfacă cerințele clientului
- Adecvat audienței

Folosirea recomandărilor de mai sus asigură întelegerea *API*-ului de către consumatorii săi. În consecință integrarea și folosirea lui vor deveni foarte ușoare. Cu cât un *API* este mai ușor de consumat cu atât rata de adopție va deveni mai mare și costul dedicat integrării sale mai mic.

### Puterea exemplului ###

Să trecem mai departe și sa punem în practică aspectele prezentate anterior prin prisma analizării unui exemplu concret de *API* împreuna cu provocările de care am avut parte de-a lungul întregului proces de design al *API*-ului. Următoarele exemple sunt luate dintr-un serviciu *web* a cărui scop este acela de a admininistra promoțiile vizibile pe *website*-ul nostru. Aceste promoții cer clientului să efectueze diverse acțiuni pe *site*(cum ar fi să se înregistreze, să puna pariuri, să se joace jocuri de tip *arcade*), toate acestea pentru a putea primi premii pre configurate.

În organizația noastra serviciile *web* comunică unele cu altele folosind un protocol *RPC*. Pentru a putea defini operațiile suportate de fiecare serviciu în parte se folosește o schema XML numita *BSIDL* (acronim pentru *Betfair Service Interface Definition Language*). Specificațiile unui *BSIDL* descriu operațiile suportate(sau metodele) pe care serviciile le expun plus tipurile de date suportate. Mai jos este un exemplu al unei operații folosite de unul dintre clienții *API*-ului ( o aplicația interna) pentru a crea o promoție.
```xml
<operation name="create">
      <description>Creates a new promotion.</description>
      <parameters>
          <request>
              <parameter name="promotion" type="Promotion" mandatory="true">
                  <description>The promotion to create.</description>
              </parameter>
          </request>
          <response type="ResponseStatus">
              <description>Return ok if it was a successful call and fail otherwise, together with an error code.
		</description>
          </response>
      </parameters>
</operation>	
```
După cum putem observa în exemplul de mai sus, numele operației este foarte sugestiv, descrie practic ceea ce operația trebuie să facă . De asemenea este foarte greu de folosit această operație într-un mod incorect, având un singur paramentru al cărui nume promotie ( *`promotion`*) este destul de sugestiv ( exprima ceea ce vrem sa creăm folosind această operație). Până în acest punct operația este foarte intuitivă.

Principala îngrijorare cu privire la noua interfață a fost cum sa modelăm o promoție astfel încat să satisfacă cerințele și în acelși timp să fie suficient de generică pentru a putea oferi flexibilitate. Cea mai simplă si usoară soluție ar fi fost să creăm un nou tip pentru fiecare promoție. Această alegere ar fi avut multe dezavantaje printre care se numară și codul duplicat - promoțiile au în comun mai multe caracteristici cum ar fi nume, descriere; rezultatul final ar fi conținut o multitudine de tipuri de promoții pentru fiecare tip nou de promoție care ar fi trebuit adăugat. Nu am fost pe deplin încântați de această variantă așa că dupa câteva sesiuni de meditat la alte soluții am venit cu idea de a avea o promoție care are toate câmpurile necesare descrierii unei promoții plus un câmp special numit *`fulfillmentCriteria`*. Un fragment reprezentând promoția modelată poate fi vazut mai jos.
```xml
   <dataType name="Promotion">
      <description>Encapsulates the promotion entity fields.</description>
      <parameter name="name" type="string">
          <description>The name of the promotion.</description>
      </parameter>
  	  <parameter name="rank" type="i32">
  		<description>Represents the promotion ranking number which is an integer on range[1,1000].
  		</description>
  	  <parameter name="product" type="set(Product)">
          <description>
              Indicates the products for which the promotion is available.
          </description>
      </parameter>
      <parameter name="fulfillmentCriteria" type="FulfillmentCriteria">
         <description>What does the user need to accomplish in order to get the reward.
  		 </description>
  	</parameter>
  </dataType>	
```
Soluția gasită cu privire la suportarea mai multor tipuri de promoții a fost sa folosim compoziția pentru a defini *API*-ul. De exemplu, în loc sa creăm un nou tip pentru promoțiile care implica un depozit(ex depozitează 10 EUR ca sa primești un pariu gratis) am creat un nou criteriu numit *`DepositCriteria`* care cuprinde cerințele necesare depozitului sumei de 10 EURO. Acest lucru face ca *API*-ul să fie unul foarte ușor de extins fară să existe schimbări incompatibile cu versiunea precedentă, beneficiind totodata de un mare avantaj, acela de a avea doar un singur tip de promoție.
```xml
<dataType name="FulfillmentCriteria">
	<parameter name="compoundCriteria" type="CompoundCriteria"/>
	<parameter name="depositCriteria" type="DepositCriteria"/>
    <parameter name="placeBetCriteria" type="PlaceBetCriteria"/>
	<parameter name="registerCriteria" type="RegisterCriteria"/>
</dataType>
```
Adăugarea tipului `CompoundCriteria` a fost necesară pentru cazurile în care o promoție este configurată sa suporte mai multe criterii în același timp folosind operatori de tip `ȘI`,`SAU` având rolul de a descrie ceea ce un client Betfair trebuie sa facă. Mai jos avem un exemplu de arbore de criterii pentru o promoție cu următoarele cerințe.

> Înregistreza-te și pune un pariu de 5 EUR pe jocurile de *Arcade* sau pune un pariu de 5 EUR pe România vs Ungaria pentru a caștiga un pariu gratis.

![](https://drive.google.com/uc?export=&id=0B8bE-oPfVra5emUwWEdoVnh6MGc)

În ceea ce privește tratarea erorilor se poate observa din primul exemplu faptul că operatia de creare a promoției returnează un tip de date numit *`ResponseStatus`* în loc sa arunce o excepție. Promoția este validată la creare și toate inconsistențeșe sunt tratate ca si erori de *business* astfel încat clientul este capabil să ia decizii bazate pe aceste erori de business. Am abordat această soluție deoarece ne-am gandit că este mai adecvată pentru consumatorul *API*-ului și așa cum spunea și Martin Fowler:
> Dacă o eroare este un comportament așteptat, atunci nu ar trebui folosite excepții.
```xml
<dataType name="ResponseStatus">
   <parameter name="success" type="bool"/>
   <parameter name="errorCode" type="string"/>
</dataType>
```
Urmarea bunelor practici prezentate mai sus nu garantează lipsa greșelilor. Noilor cerințe care au apărut au cauzat adăugarea de noi atribute necesare descrierii unei promoție. 
```java
Promotion
  getPromotionName()
  getRank()
  getProduct()
  getDisplayAttributes()
  getJurisdiction()
  getVisibleForGuest()
  getTermsAndConditions()
```
Unele atribute au fost grupate în noi tipuri de date cu nume sugestive.
```java
DisplayAttributes
  getTheme()
  getBannerUrl()
  getImageUrl()
```
Alte atribute au fost adăugate direct fară sa fie grupate pentru că la momentul respectiv nu am găsit un grup potrivit iar adăugarea unui singur atribut într-un grup nu ar fi avut sens. Ca și consecință în aceast moment există foarte multe atribute în tipul promoție (*`promotion`*) și chiar dacă le-am putea adauga într-un grup potrivit (am putea grupa *`rank`* și *`product`* într-un grup similar cu cel pentru *`DisplayAttributes`*), unul dintre clienții *API*-ului folosește deja versiunea cu atributele negrupate făcând astfel dificil procesul adaugării unor modificări incompatibile cu versiunea anterioară.
	
### Concluzii ###
Definirea unui *API* este o activitate captivantă. Ce am învațat noi din aceasta experiență este câ deși nu toate schimbările de cerință pot fi anticipate din timp, este bine să incerci să prevezi ceea ce va urma, pentru că în acest fel *API* ul va deveni mult mai ușor de extins și vor fi și mai putine modificari incompatibile cu versiunea anterioară. Alt lucru pe care l-am învațat din acesta experiența a fost acela că nu tot timpul putem face din prima lucrurile bine, dar este important să fim capabibili sa conștientizăm problema și să gasim soluția adecvată.