# Chwyt — grafik pracy

Responsywny prototyp aplikacji do planowania pracy ściany wspinaczkowej. Frontend: Vue 3, TypeScript i Vite.

## Uruchomienie

```sh
npm install
npm run dev
```

Produkcja: `npm run build`.

## Co zawiera prototyp

- kalendarz miesięczny z przykładową obsadą;
- widok zespołu i sumę godzin na pracownika;
- ręczne dodawanie dyżurów i zajęć, z minimum 4 godzinami na wpis;
- składanie i rozpatrywanie próśb grafikowych;
- propozycję dyżurów dla godzin otwarcia;
- polskie święta ruchome i stałe, z pominięciem ich przez generator;
- lokalny zapis w przeglądarce.

## Zakres, który wymaga backendu

Dane są przechowywane tylko w bieżącej przeglądarce. Logowanie, współdzielenie danych między pracownikami, trwała baza PostgreSQL, reguły zakładki zajęć, kontrola prawa pracy oraz Google Calendar nie są jeszcze podłączone. Wygenerowany grafik jest propozycją i wymaga kontroli przed publikacją.
