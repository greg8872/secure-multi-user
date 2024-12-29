# secure-multi-user
Secure stored data for allowing multiple users to access/edit encrypted data

NOTE: I'm still learning to make projects on GitHub (old dog learning new tricks), so formatting not up to snuff on here, (and some things are applying unwanted formatting), so for now, you will want to view this in RAW format. Sorry.

This is just a preliminary experiment to allow data to be stored encrypted, but allow multiple users to use that data. 

I would prefer this to include a hash that comes from the users password, however if a user needs to do forget password, we cannot regenerate keys for them with the new password as we don't have the old password to get the keys in the first place. Best I could do would be to have a "global" key store (similar to a user with a password), but this would require that password to be on the server as well, so in the end, I just went overboard with storing the keys with the user (or other table defined in "use")

We allow for multiple "use" (table suffix) as for the system that I was making this for allowed for people who are not yet users to submit data, so we can't store strictly off users (we have an "onboarding" table that is keyed off the users e-mail address)

This is NOT recomended to be used for any projects. This is EXPERIMENTAL ONLY. USE AT YOUR OWN RISK.

===================================================================================

Tables needed for this system: (suggest to be modified to have a prefix)

In any table that will be a type of storage, example here is "users", note these are the fields specific for this use, you will obvioustly want to add others.

Users: 

id - unsinged into autoincrement
tsCreate - datetime (set when the user was created)
hash - text (will store json encoded array of 32 items of 16 character random keys)

Keys: (this will contain the key for the enctyped data, encrypted for each use to access)

id - unsinged int autoincrement
tsCreate - datetime (set when created)
tsModify - datetime (my DB class auto updates)
use - enum['users','oboarding'] (list of table names (after any prefix) this key is stored for) 
useId - unsigned int (the id from the use table that owns the key)
secureId - unsigned int (the id of the actual encrypted data store)
revision - unsigned int (raw timestamp of when the key was stored. (for this table, it is basically tsCreate)
key - text (the key for data store that is encrypted for this use)

Secure: (this is the table that will contain the data enctypted)
id - unsinged int autoincrement
baseId - unsigned int (for "revisions", this is the id it is a revision of. 0 = This is the 'live' version)
tsCreate - datetime (set when created)
tsModify - datetime (my DB class auto updates)
ipCreate - varchar[16] (the IP of who created the store)
ipModify - varchar[16] (the IP that modified the store)
use - enum['users','oboarding'] (list of table names (after any prefix) this key is stored for) 
useId - unsigned int (the id from the use table that owns the key)
stats - enum['active','inactive','archived'] (status of the store. anything 'archived' auto deletes after a month per project cron)
revision - unsigned int (raw timestamp of when the key was stored. (for this table, it is basically tsModify)
data - text (the actual encrtyped data)

===================================================================================

Things to program (this is a live "notes for me as I build it")

private uses = ['users','onboarding'] - the list of tables allowed for "use" (may eliminate this, as it makes the class set to the project, where idealy it should be portable for other uses)

private falseValue = '~!#FALSE#!~' - If data passes in as an actual FALSE value, encrytpe this string instead, and on decrypting, convert this back to FALSE
private nullValue = '~!#NULL#!~' - Same as above but for NULL 

static function genreateHashes() - to get an array of 32 string of 16 random characters in length to be JSON encded into the user (or other use) tables 'hash" field

class constructor - For when making an object, preliminary items passed (may rework) This is per "Use"/"UseId", 
  use,
  useID,
  hash, (from use table)
  userLevel (0-255, if the use is a user, the userlevel, otherwise 0) - Only needed for cross checking values before returning

public function storeData - Store the data 
  storeId, - The ID of the existing store if you are modifying one - 0 = new store
  adminLevel, - (optional [default 255]) The minimum level of admin to create keys for so admin can edit/see data as well (note, for my projects, access levels are 0-255, 0 - no login, 255 - super admin user)
  extraUseIds, - (optional [default empty]) A list of other useIds who should have a key (ie, you have sub users), stored as (usetype)(useid), ex ['users343','onboarding22'] 
  revisions, - [true/false] Auto create revisions if storing to an existing value
  data, - the data to actuall store in RAW format.
  RETURNS array of status: 
  [
    status: 'success' or 'error',  (if error, all other items are FALSE)
    id: the id of the store
    revision: the revision of this store
    key: the radomly genreated key used for this store 
  ]

public function readData - Read a record 
  storeId, ID of the data to retrieve
  revision, optional revision # to get (for "history" of saves)
  RETURNS array of status: 
  [
    status: 'success' or 'error',  (if error, all other items are FALSE)
    data: the raw data that was originally encrypted
  ]
   
===================================================================================

Notes for data on store...

this is the dataformat before enctyped:
  revision
  use
  useid
  json_encode(extraUseIds)
  adminLevel
  json_encode(data)

When decrypting data, we make sure that:
  it starts with numbers that are the same as the revision
  it then has [a-z_] for the use
  it then as a number that is an id of the use
  it then has a json_decode(x,true) that is valid
  it then has a numebr that is 0-255 for the min admin user levels
  the rest needs to properly json_deocde(x,true)

  return data IF:
    {
        use/useId set on contructor match the values from decryption
      OR
        use/useId set on consteuror is in the list of extrauseIds
      OR 
        userlevel set on constructor is at least level specified for adminLevel
    )
    AND
      data does not properly json_decode into something

  
