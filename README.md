# FitMeIn

Staying healthy is a big deal, but let's be real – keeping up with it can be a bit overwhelming, especially when it comes to breaking a sweat. That's where FitMeIn comes in! If you're struggling to get yourself moving, don't worry – you're not alone. FitMeIn is all about making exercise more fun and finding friends along the way.

Here's how it works: just pick an activity you actually enjoy, and we'll hook you up with a workout buddy in your area who's into the same stuff. It's like having a fitness pal to cheer you on (and hold you to your word)! We're all about bringing people together, not just to get fit but to make some connections along the way. Let's crush those fitness goals and make some memories, shall we?


##
**Team** <br><br>
This was a group project in week 9 of the General Assembly Software Engineering Immersive Bootcamp. We had 7 days to complete the project, tasked with architecting, designing, and collaboratively constructing a full-stack web app using Python and Django.
Working as a team to build this app taught us how to pair and mob program and practice the pull/push workflow within a group.<br><br>
Meet the Team: [Angelica Sandrini](https://github.com/AngelicaSandrini) | [Hannah Curran](https://github.com/HannahCurran) | [James Carter](https://github.com/JamesCarter10) | [Lucas Neno](https://github.com/LucasNeno)

So here it is, our [FitMeIn app](https://fitmein-d13b6d574438.herokuapp.com/)!

##
**Technology Used** <br><br>
**HTML**   | **CSS**   | **Bootstrap**   | **JavaScript** | **Python** | **Django** | **Git** | **GitHub** | **PostgreSQL** | **Heroku** | **Amazon AWS**

##
**Brief**
- Connect to and perform data operations on a PostgreSQL database (the default SQLite3 database is not acceptable).
- If consuming an API, have at least one data entity (Model) in addition to the built-in User model.
- If not consuming an API, have at least two data entities (Models) in addition to the built-in User model.
- Have full-CRUD data operations across any combination of the app's models (excluding the User model).
- Authenticate users using Django's built-in authentication.
- Implement authorization by restricting access to the Creation, Updating & Deletion of data resources.

##
**Planning** <br>

**Brainstorming** <br>
![text](https://github.com/hannahcurran/fitmein/blob/main/fitmein%20planning.png)


**Wireframe** <br>
![text](https://github.com/hannahcurran/fitmein/blob/main/fitmein%20mobile%20wireframes.png) 
<br>
![text](https://github.com/hannahcurran/fitmein/blob/main/fitmein%20wireframes.png)

**ERD** <br>
![text](https://github.com/hannahcurran/fitmein/blob/main/fitmein%20ERD.png)

##
**Code Process**
<br>
During the project, we spent some time pair programming, mob programming and we also had specific areas of work to complete individually. My primary focus within the project was work on the user's profile page, including the implementation of CRUD functionality. 


``` python
@login_required
def profile(request):
    try:
        profile = Profile.objects.get(user=request.user)
        context = {'profile': profile}
        if profile:
            profile_form = ProfileForm(instance=profile)
            comments = Comment.objects.filter(user=request.user)
            context['profile_form'] = profile_form
            context['comments'] = comments
        return render(request, 'user/profile.html', context)
    except Profile.DoesNotExist:
        return redirect('create_profile')


# ---------------- Sign-Up ------------------------

def signup(request):
    error_message = ''
    if request.method == 'POST':
        form = UserCreationForm(request.POST)
        if form.is_valid():
            user = form.save()
            login(request, user)
            return redirect('create_profile')
        else:
            error_message = 'Invalid sign up - try again!'
    form = UserCreationForm()
    context = {'form': form, 'error_message': error_message}
    return render(request, 'registration/signup.html', context)


# -------------Create Profile----------------

class ProfileCreate(CreateView):
    model = Profile
    template_name = 'user/create_profile.html'
    fields = ['age', 'gender', 'location', 'is_couch_potato', 'favorites', 'latitude', 'longitude', 'is_active']
    success_url = reverse_lazy('profile')  # Replace 'profile-detail' with your actual URL pattern

    def form_valid(self, form):
        print('form_valid being executed')
        form.instance.user = self.request.user
        print(form)
        return super().form_valid(form)

```

**API** <br>
Amongst one of the features is the API call which Lucas took the lead on: our program matches two or more users using their GPS/IP location by means of the HTLM5 Geolocation API. When first prompted, the user provides data on which activities they would like to perform and this information is stored in their profile. The program then makes a JavaScript call to the API that takes the user's 'latitude' and 'longitude', it then passes on those values through an URL to the back-end which are then parsed and stored in the database. Finally, a filter is applied to every active profile in the database that has selected similar activities to that of the user, and calculates the distance between these profiles and the user, returning a table of profiles within a certain range that have the same activity interests as our user.

``` python
const displayCoord = document.getElementById("displayCoord")
const latitudeDisplay = document.getElementById("latitude")
const longitudeDisplay = document.getElementById("longitude")

displayCoord.addEventListener("click", getLocation)

function getLocation(event) {
    event.preventDefault()
    console.log('click')
    if ("geolocation" in navigator) {
        navigator.geolocation.getCurrentPosition(success, error);
    } else {
        alert("Geolocation is not available in your browser.");
    }
}

function success(position) {
    const latitude = position.coords.latitude;
    const longitude = position.coords.longitude;
    console.log('success')
    console.log(latitude)

    window.location.assign(`http://localhost:8000/my_match/${latitude}/${longitude}/`)

}
```

``` python
def haversine(lat1, lon1, lat2, lon2):
  R = 6371
  dist_lat = math.radians(lat2 - lat1)
  dist_lon = math.radians(lon2 - lon1)
  a = math.sin(dist_lat / 2) * math.sin(dist_lat / 2) + math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) * math.sin(dist_lon / 2) * math.sin(dist_lon / 2)
  c = 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))
  distance = R*c
  return distance

def find_match(request, profile_id):
  profile = Profile.objects.get(id=profile_id)
  user_latitude = profile.latitude
  user_longitude = profile.longitude
  user_chosen_activities = profile.chosen_activities
  active_profiles = Profile.objects.filter(is_active=True, chosen_activities__contains=user_chosen_activities).exclude(id=profile.id)
  matched_profiles = []
  matched_distance = [] 
  #Check if haversine distance is within a range (5.0km)
  for profile in active_profiles:
    distance = haversine(user_latitude, user_longitude, profile.latitude, profile.longitude)  
    if distance < 5000.0:
      matched_profiles.append(profile)
      matched_distance.append(distance)
  print(matched_profiles)
  print(matched_distance)
  return render(request, 'user/my_matches.html', {'profile':profile, 'matched_profiles':matched_profiles, 'matched_distance':matched_distance, 'user_latitude':user_latitude, 'user_longitude':user_longitude})
```

##
**Challenges** <br>
Some of the most challenging moments during this project revolved around the pull/push and migrate aspect of team work and PostgreSQL. We frequently had conflicts in the beginning and dedicated a lot of time to debugging and resetting the database. Also, our final relationship between entities wasn't correct so our model was flawed and when testing, we weren't able to relate the user's unique ID to the comment section. Also, as the functionalities we wanted to implement were different from what we did during class and labs, that meant that we all had to go through a lot of documentation in order to implement them, causing us to use a lot of our time studying.

##
**Wins** <br>
During our project, we successfully achieved crucial milestones as a team. We introduced several features and now have a fully operational app. Despite facing challenges, we had good communication and everyone brought a patient and supportive outlook which helped to pull each other through. Through teamwork and open communication, we laid the groundwork for a great social fitness app!

##
**Key Learnings** <br>
We've definitely gained a lot of experience in Git and GitHub collaboration throughout the hours invested in it! I've learned how clear, regular communication throughout projects is really important. It was great to have the opportunity for pair and mob programming at times, as there's so much to learn from each other and we learned how important it can be for boosting morale during some of the trickier times. Plus, dealing with real-world challenges like adding authentication, an API, and balancing front-end aesthetics with back-end functionality enhanced our technical skills in different ways.

##
**Future Developments**<br>
- At the moment there’s a bug with the photo upload (it uploads to all users and once uploaded cannot be changed) and the comments are not linked solely to the profile for which they are added.
- We would like the app show all the connections made and improve its social feature.
- We would like it to also be a place where, based on the user's location, fitness events in the area can be suggested.
- We would like to add the possiblity to also find recipes and eating habits and suggestions to improve the user's health.
- We finally but not lastly would like to improve its style and UI/UX.





