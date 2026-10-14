# Felsökningsrapport – överlappande bokningar

## Felet

Jag undersökte ett fel som gjorde att systemet kunde godkänna överlappande bokningar för samma rum och datum. Vissa överlappningar fångades och godkändes inte, men bokningar som började före en existerande bokning och slutade under eller efter den godkändes.

## Återskapning av felet

Felet kan återskapas genom att först skapa en bokning för ett rum mellan klockan 09:00 och 11:00. Sedan försöker man skapa en ny bokning för samma rum och datum mellan 08:00 och 10:00. De andra uppgifterna måste vara korrekta för att bokningen ska nå överlappningskontrollen.

## Resultat

Den gamla koden godkände den nya bokningen. Det var fel eftersom båda bokningarna använde rummet mellan 09:00 och 10:00.

## Förväntat resultat 

Det förväntade resultatet var att den nya bokningen skulle nekas med meddelandet `The room is already booked during the selected time.`

## Var felet fanns 

Felet fanns i metoden `Create` i `BookingService.cs`:

```csharp
booking.StartTime >= existing.StartTime &&
booking.StartTime < existing.EndTime
```

## Anledning till felaktigt resultat

Den gamla kontrollen undersökte om den nya bokningens starttid låg inom den andra existerande bokningens tidsintervall.

För bokningen 08:00–10:00 blev den första jämförelsen falsk, eftersom 08:00 inte är senare än eller lika med 09:00. Därför upptäcktes ingen överlappning.

## Rättningen

Jag ändrade villkoret till:

```csharp
booking.StartTime < b.EndTime &&
booking.EndTime > b.StartTime
```

Nu kontrolleras att den nya bokningen börjar före den existerande bokningens slut och slutar efter dess start. Om båda villkoren uppfylls så överlappar bokningarna. Kontrollen gäller fortfarande bara aktiva bokningar för samma rum och datum.

## Verifiering

Jag verifierade att ändringen fungerade genom att skapa en bokning mellan klockan 09:00 och 11:00. Sedan skapade jag en ny bokning för samma rum och datum mellan 08:00 och 10:00. Systemet visade felmeddelandet: `The room is already booked during the selected time.` Jag kontrollerade också att den nekade bokningen inte sparades. Jag verifierade även att ändringen fungerade genom att skapa en bokning för samma rum och datum mellan klockan 11:00 och 12:00. Den bokningen godkändes och sparades.

Jag verifierade även att ändringen fungerade genom att köra `dotnet test` och kollade att testerna: `Create_WhenBookingOverlapsExistingBooking_ReturnsFailure` och `Create_WhenBookingStartsWhenExistingBookingEnds_ReturnsSuccess` passerade.

## Automatiserade tester som skyddar mot felet

Testet `Create_WhenBookingOverlapsExistingBooking_ReturnsFailure` använder en existerande bokning mellan 09:00 och 11:00 och kontrollerar fyra olika överlappningar. Där ingår 08:00–10:00 och 08:00–12:00, som den gamla koden missade. Testet kontrollerar både att bokningen nekas och att rätt felmeddelande returneras.

Testet `Create_WhenBookingStartsWhenExistingBookingEnds_ReturnsSuccess` använder en existerande bokning mellan 09:00 och 11:00 och kontrollerar att en bokning mellan 11:00 och 12:00 godkänns. Det kontrollerar att bokningar som ligger direkt efter varandra tillåts.

Testerna skyddar både mot att överlappningsfelet återkommer och mot att kontrollen blir för strikt.