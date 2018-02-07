<table>
  <tr>
    <td><a href="service">&lt; 2.3. Service Implementation</a></td>
    <td align="right"><a href="riak-converter">2.5. Riak Converter &gt;</a></td>
  </tr>
</table>
# 2.4. Data Access Object

* Extension Point: Override CRUD Method

----------
__Extension Point: Search field name and Search field value__

[Riak Solr](http://docs.basho.com/riak/latest/dev/using/search/) (which is used by the Search APIs) by default suffixes the Riak Datatype to the field name. For ex: *_set, *_register, *_map, *_counter etc. The framework takes care of these field names in the search functionality. But there may be times when you have stored your field differently / you have complex resource structure and need complex processing. During all this, if there are any @Searchable fields, then you will have to take care of constructing the search field name and search field value, which will be used to hit Riak Solr. 

The framework provides 2 methods for this purpose, which can be extended for additional functionality:
* `#getSearchFieldName`
* `#getSearchFieldValue`

Call the `#super` methods in each of these to take care of most of the processing that the framework will give you for free. And then add any of your custom processing logic. You can see an example of a resource DAO class which has these extensions [here](http://git.covisintrnd.com/eng-iot-core/device-service/blob/master/server/src/main/java/com/covisint/platform/device/server/device/DeviceRiakDao.java).

----------

<table>
  <tr>
    <td><a href="service">&lt; 2.3. Service Implementation</a></td>
    <td align="right"><a href="riak-converter">2.5. Riak Converter &gt;</a></td>
  </tr>
</table>