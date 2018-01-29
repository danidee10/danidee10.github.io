<h4 class="text-underline">ATMFinda: </h4>

ATMFinda was my attempt at solving the annoying issue of non-functional ATM's in Nigeria. It's pretty common to go to an ATM and after inserting your card, inputting your password you get the dreaded "Temporarily unable to dispensh cash" message.

The idea was to create a mobile app that pulls data from an API of ATM's around the country (Provided by Banks) about the status of various ATM.
Using google places API and Google maps we could easily display a custom maps showing Active ATM's (green) and Inactive ATM's (Red).

It failed because for some reason, no bank listened to us (I had other people push the idea to them). So we didn't have any API to use.

I decided to crowd source the app but to be honest it wasn't feasible No one has the time to click around to say "Hey this ATM is active/inactive" when there's no reward or points to gain. I also thought about implementing Background Geolocation like Google maps but that didn't work because of a limitation in the [Expo framework](expo.io). At the moment it's impossible to have background geolocation with Expo without Ejecting the application and building it manually.

The good news is that the Expo team [is working on background geolocation](https://expo.canny.io/feature-requests/p/background-location-tracking) and i think it should be ready sometime in 2018.

ATMFinda was built with React Native with Flask serving as the backend API. The source code for the Server and the Mobile app is available publicly on github.

URL: [ATMFinda on Expo](https://expo.io/@danidee/atmfinda)

Public Repos:

- [atmfinda api](https://github.com/danidee10/atmfinda-api">atmfinda-api)

- [atmfinda mobile app](https://github.com/danidee10/atmfinda)

Technologies:
<button class="btn btn-ghost tag">python</button>
<button class="btn btn-ghost tag">JavaScript</button>
<button class="btn btn-ghost tag">flask</button>
<button class="btn btn-ghost tag">react-native</button>
<button class="btn btn-ghost tag">google-places-api</button>
<button class="btn btn-ghost tag">google-maps</button>

<a class="project-link" href="../images/about/projects/atmfinda.png" data-lightbox="atmfinda">
<img src="../images/about/projects/atmfinda.png" />
</a>
<a class="project-link" href="../images/about/projects/atmfinda2.png" data-lightbox="atmfinda">
<img src="../images/about/projects/atmfinda2.png" />
</a>
<a class="project-link" href="../images/about/projects/atmfinda3.png" data-lightbox="atmfinda">
<img src="../images/about/projects/atmfinda3.png" />
</a>
<a class="project-link" href="../images/about/projects/atmfinda4.png" data-lightbox="atmfinda">
<img src="../images/about/projects/atmfinda4.png" />
</a>
<a class="project-link" href="../images/about/projects/atmfinda5.png" data-lightbox="atmfinda">
<img src="../images/about/projects/atmfinda5.png" />
</a>
</p>