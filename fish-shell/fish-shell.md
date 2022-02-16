# De fish-shell

Het eerste dat ik installeer op een nieuwe laptop? 
De *fish*-shell. 
Je went enorm snel aan handige features als suggesties uit je shellhistorie, tab completion op o.a. Git-commando's en syntax highlighting. 
Ook de web-based configuratie is erg gebruiksvriendelijk!

## Fish?

'Fish' staat voor **F**riendly **I**nteractive **SH**ell en is een command-line shell die zich voornamelijk richt op gebruiksvriendelijkheid, waarbij de expressiviteit niet in het geding mag zijn.
Naast gebruiksvriendelijkheid is een ander belangrijk ontwerpprincipe dat in fish alles mogelijk moet zijn wat ook in andere *shells* mogelijk is.

## Automatische suggesties

Terwijl je typt, suggereert fish commando's rechts van je cursor. 
De shell put hiervoor uit bestaande bestandspaden, commando-opties en uit je shell-historie. 
Met `→` of `Ctrl + F` kun je de suggestie accepteren. 
Met `Alt + →` accepteer je de suggestie per woord. Of typ gewoon verder om de suggestie te negeren. 

## Tab completion

Fish genereert de *completions* automatisch door de geinstalleerde *man pages* van tevoren te parsen.
(...)

Dit is voor de makers van fish een belangrijke feature, omdat één van hun doelen is om nieuwe gebruikers snel experts te laten worden. 
En een belangrijk criterium voor de snelheid van dat proces is *discoverability*: de mate waarin een nieuwe gebruiker de mogelijkheden van een softwareprogramma zelf kan ontdekkken.
Bij een programma met een grafische interface is de *discoverability* vaak hoog, omdat de mogelijkheden direct zichtbaar zijn in de interface.
Bij een command-line interface is de *discoverability* een stuk lager, omdat de mogelijkheden verstopt zitten in de *man pages*, en die moet je als gebruiker zelf weten te vinden om een expert te worden.
Door de *man pages* aan te bieden via *tab completions* krijgen nieuwe gebruikers in een kortere hoeveelheid tijd meer mogelijkheden te zien.

## Syntax highlighting

Fish geeft ongeldige commando's of niet-bestaande bestandslocaties in rode tekst weer. 
Een geldig commando heeft standaard groene tekst, terwijl een geldige bestandslocatie wordt onderstreept. 

## Web-based configuratie

Deze standaardopmaak kun je wijzigen door `fish_config` uit te voeren. 
Naast opmaak kun je hier ook functies en aliassen instellen.

## Uitbreidingen

* Starship
* Nerd Fonts
* oh-my-fish

## Is de fish-shell iets voor mij?

De fish-shell past goed bij je wanneer je al wat ervaring hebt met *bash*, maar daarin soms wat features mist (zoals automatische suggesties). 
En ook als je nu al wat meer van je shell eist (ik denk dan vooral aan gebruikers van *zsh* en mogelijk ook *oh-my-zsh*), is Fish het proberen waard. 
Fish kan namelijk alles wat *zsh* ook kan (inclusief de uitbreidingen), maar zorgt er daarnaast voor dat je minder handmatig hoeft in te voeren.

Wat besturingssystemen betreft is Fish beschikbaar voor Linux en macOS. Op Windows 10 kun je het draaien binnen het Windows Subsystem for Linux. Voor oudere Windows-versies zul je Cygwin of MSYS2 nodig hebben.

## Referenties

* [Website](https://fishshell.com/)
* [GitHub](https://github.com/fish-shell/fish-shell/)
* [Fish tutorial](https://fishshell.com/docs/current/tutorial.html)
* [Fish design principles](https://fishshell.com/docs/current/design.html)
