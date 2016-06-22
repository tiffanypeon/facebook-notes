# Happy Path Ad Publishing (aka where stuff usually goes wrong)

There are a lot of things that need to happen in order for ads to make it all the way to successful activation (and even then, more dependencies involved in reporting and successfully transitioning an ad to ended). Here's an overview of what needs to go right, and some notes about where things usually go wrong.

## Publishing an ad from Boost to FBAA -
- When a user buys a Boost Ad, their ad isn't necessarily ready to get sent to FBAA right away
- When the user hits the purchase button, we do a bunch of stuff synchronously:
```ruby
# charge campaign service class is called from the campaigns controller
class ChargeCampaign
  def call
    set_campaign_attrs
    save_and_charge_account
    campaign.update_terms_of_service
    BoostActivity.new(campaign).generate
    ...
```
- If any of these calls fail (with the exception of Boost Activity generation), the charge is wrapped in a transation and the user will see an error
- If this is successful, the user will see a success message, and the rest of the tasks are performed in the background
- The first thing we do is make an async call to claim the facebook page on behalf of our user (more info in the "possible failures" section of debugging-tools)

**PublishReadyCampaignsJob** -
- The rest of the work is taken over by this job, which iterates through every campaign with a status of "new" and checks to see if it is ready to be sent to FBAA
- Three things need to happen in order for a campaign to be considered ready for delivery:
  - The email campaign must be sent: Boost ads are Facebook link ads, and therefore need a live link for the ad to link to. Because we don't create that link until after the email is sent, we need to wait for the email to go out. **Problem area** Customers who unschedule their email will never have a live link for ads and therefore their ad will never get sent to FBAA.
  - The permalink must be visible: The permalink is the link generated when the email is sent. **Problem area** If the link is down for some reason, there is a check that will stop the ad from being sent to FBAA.
  - The boost activity status must be created: The activity status is something used by internal CTCT services and I don't fully know what they use it for. We make one final check here to verify that we created it (we also try upon charge of the campaign). I've never seen it fail twice, so this isn't really a problem area.

### How do I figure out what's going wrong?
- Log onto [backstage](p2-prod-boost-02.ctct.net:8080/backstage/) and tail your logs:
```
ssh p2-prod-boost-02.ctct.net
tail -f /opt/cc/logs/server_boost.logs | grep 'SOME IDENTIFIABLE STRING'
```
- Use the methods called within CampaignVerifier#verify_activity_statuses to see which call is returning false

## Publishing an ad from AdLauncher:
- Currently, this is done synchronously, so all errors will be surfaced in the admin console

## Once ads get to FBAA:
I find it easiest to think about the publish flow in four stages:

### AdAccount creation:
Creation happens in the first call in the PublishAd service class. All it does is create a new ad account for our user if they haven't advertised with this facebook page before(uniqueness is determined by the facebook page id), set it up so that we are in charge of the account (our customers have no connection to the ad account), and connect them to our credit line so that we are invoiced for the charges incurred by their ads.

### AdAccount set up:
Set up is a combination of assigning the correct API users and dealing with some legal requirements from FB.

- Every ad account and facebook page need to have our API user (BoostDevUser) assigned to it in order for us to make calls on behalf of that object (happens in the UserPermissions class) **Problem area** If a Boost was created a while ago but wasn't sent for some time, there's a chance the user changed his/her FB password and that we lost the permissions we were granted when the Boost was created. There are directions for MO to fix this error in FBAA>
- Every ad account needs to accept terms of service for custom audience and lookalike audience permissioning (happens in the TermsOfService class)
- Lead ad accounts need to also accept the lead ad terms of service (happens in the TermsOfService class) **Problem area** If the ClaimFacebookPage calls failed silently in AdLauncher, this call will fail with a very unhelpful message. Try recalling that service in the AdLauncher service and then republishing.
- Lead ad accounts also have to set up a privacy url page for the ad account (happens in the LegalContent class) **Problem area** This is a beta API and will likely change requirements without notification from FB. If it breaks, contact our tech resource.

### Audience creation and population:


### Ad creation:
