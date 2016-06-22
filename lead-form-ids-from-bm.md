# Finding old lead form ids:

The Leads API doesn't currently offer an endpoint that allows us to look up all of the ad form ids that have ever been created for an ad.

**When is this a problem?**
The Leads API also doesn't support updating an existing ad form, we have to do the same thing we do with ad creatives, which is to create a brand new ad form and overwrite the AdForm#facebook_id with the new id. There is an edge case where if for some reason the leads for an old ad form weren't correctly syncing into the account, once the ad form is updated we no longer check that ad form for new leads. Because we no longer have the id of the old ad form, in order to solve this problem we need to temporarily switch out the AdForm#facebook_id and manually trigger the GetLeadsFromAdForm service to sync the old leads.

## Where do I find old lead form ids:

### Short answer: In business manager

### Long answer:
- Add yourself to the customer's page in business manager (this will allow you to see the ad forms)
- Add yourself to the customer's ad account in business manager, and then go into their ads manager
- Click through to the ad group and click to edit the ad
- You will see a dropdown with the names of all available ad forms (don't see them? Make sure you added yourself to their Facebook page)
- Inspect element and select through each option to see the value of the dropdown option change, showing the facebook_id of each form

## Now what?
- You want to figure out which ad forms have leads associated with them.
- For each id you've recorded, make the koala call to see if that ad form has leads
```ruby
  KoalaWrapper.new(
    facebook_id: [the id you're testing],
    endpoint: 'leads',
    params: { limit: 1000 }
  ).get
```
- Keep track of which ids have leads
- Copy the original ad form id so that you can reassociate it after forcing all of the old leads through (ad.ad_form.facebook_id)
- Update the ad.ad_form.facebook_id with the first id in your list of ids with leads
- ` GetLeadsForAdForm.new(ad).call `
- Repeat until you've reached the end of your list of ids with leads
- Update the ad.ad_form.facebook_id back to your original facebook_id so that moving foward all new leads will get synced
