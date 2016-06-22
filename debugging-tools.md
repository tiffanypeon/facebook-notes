# Facebook debugging tools:

- [Graph API explorer tool](https://developers.facebook.com/tools/explorer/145634995501895/) - Sometimes requests we make will fail due to improperly formatted parameters. Because Koala is doing some pre-formatting, using this tool is the easiest way the problem is/isn't related to our Ruby wrapper. You can also use a CURL from the terminal as it now returns the fb reference id that support usually asks for to identify bugs.

- [Testing lead ads](https://developers.facebook.com/tools/lead-ads-testing) - You can use this testing tool to add a single lead to your ad form. If you have existing leads it will delete them, so don't use this on production data!

# Filing support tickets:

- [Direct support portal](https://developers.facebook.com/direct-support) - When there appears to be a bug with a non-beta API endpoint, any general questions about a failure with a single account, etc, this is the fastest way to get a response. Provide the auth token, steps to recreate, and when possible, the fb reference ID in order to expedite the process. They respond within 24 hours. Steps to file a bug:
  - click 'Ask a question'
  - select 'Marketing Parner: Technical Support' from the dropdown
  - Type the bug name and click 'Ask a New Question'
  - Fill out recreation steps, etc, in next window

- Engineering contact - Our current engineering contact is Jay Yan (jayy@fb.com). He's a developer on the beta lead ads API. Email him to expedite high priority bug resolution (send him an email with the direct support ticket link) or for any questions on undocumented endpoints. He only responds to the most recent email, so try to consolidate all emails into one rather sending a thread.

- [Generic support](https://developers.facebook.com/bugs/) - If your issue is a legitimate bug, direct support will open a ticket here. The support is slower and spotty. Get everyone on the team to subscribe to the bug and ping Jay and our product contact with the link to expedite.

# Common issues:

- Secret JSON requirement: Facebook often writes in documentation that a parameter should be an certain data type, but what they really mean is it should be the json representation of that data type. If you get back an error message like, "X parameter must be an array" when you are passing in an array, try calling .to_json on that parameter. It usually works and will save you several hours of digging into Koala's middleware.

- An endpoint is down: If all of the ads in FBAA start failing, it's likely an endpoint has broken. This happens several times a year and takes around 24 hours to get resolved. File a ticket with direct support and then will likely link you to the existing issue. Subscribe, ping all contacts, and wait.

- Failed to add page to business manager: I think there's a bug ticket for this, but on the client-side code, we have to make a couple of Facebook calls because we need to use the user's token and don't want to be passing that between services. Here are the calls that we make (and their purpose) for each client:
  Boost:
    - Claiming the Fb page to business manager: We make a single call to the /[page_id]/agencies endpoint to add the page to our business manager. We can do this without any customer assistance because we have the user's token. Adding the page to our business manager allows us to publish the ad creative associated with that page and gives us access to ads that display in the middle of the newsfeed.
  AdLauncher:
    - Claiming the Fb page to business manager (same as above)
    - Adding the system user to the Fb page. We usually do this in FBAA, but we need to have the user added to the page in order to whitelist the user for reading lead ads.
    - Whitelisting the system user to read ads - we need the user token to do this, if the user isn't whitelisted when we try to read the leads we'll get back an empty array

  Anyway, back to the failing. Because we make these calls in a background thread we don't raise any sort of error. If this fails it will show up as a permission error in FBAA when we are trying to assign the users to the Facebook page or trying to publish the AdCreative, depending on where in the process the failure occurred. It can be easily remedied by making the claim page call again in the console of whichever client the ad came from and then republishing the ad. You can verify this issue by going into business manager and seeing that the page is not claimed in BM.
