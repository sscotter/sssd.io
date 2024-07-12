Common AD Provider issues
#########################

* How do I set up the AD provider?

  * There is a dedicated page about AD provider setup

* Many users can't be displayed at all with ID mapping enabled and SSSD domain logs contain error message such as:

    .. code-block:: sssd-log

        [sdap_idmap_sid_to_unix] (0x0080): Could not convert objectSID [S-1-5-21-101891098-1139187330-4192773280-XXXXXX]

    * If you are running an old (older than 1.13) version and XXXXXX is a number larger than 200000, then check the ``ldap_idmap_range_size`` option. You'll likely want to increase its value. Keep in mind the largest ID value on a POSIX system is 2^32.

    * If you are running a more recent version, check that the ``subdomains_provider`` is set to ``ad`` (which is the default). Some users are setting the ``subdomains_provider`` to ``none`` to work around fail over issues, but this also causes the primary domain SID to be not read and therefore cannot map SIDs from the primary domain. Consider using the ``ad_enabled_domains`` option instead!

* The POSIX attributes disappear randomly after login

  * SSSD looks the user's group membership in the Global Catalog to make sure even the cross-domain memberships are taken into account. Chances are the POSIX attributes are not replicated to the Global Catalog. You can disable the Global catalog lookups by disabling the ``ad_enable_gc`` option, but you'll lose cross-domain memberships. Alternatively, modify the AD schema to replicate the POSIX attribute to the Global Catalog.

* After selecting a custom ``ldap_search_base``, the group membership no longer displays correctly.

  * If you use a non-standard LDAP search bases, please disable the TokenGroups performance enhancement by setting ``ldap_use_tokengroups=False``. Otherwise, the AD provider would receive the group membership via a special call that is not restricted by the custom search base which causes unpredictable results
  * Typically, users configure a custom ``ldap_search_base`` to limit the groups the user is a member of. Please see `this blog post <https://jhrozek.wordpress.com/2016/12/09/restrict-the-set-of-groups-the-user-is-a-member-of-with-sssd/) for more information on the subject>`_.

* SSSD keeps connecting to a trusted domain that is not reachable and the whole daemon switches to offline mode as a result

  * SSSD would connect to the forest root in order to discover all subdomains in the forest in case the SSSD client is enrolled with a member of the forest, not the forest root. This is because only the forest root knows all the subdomains, the forest member only knows about itself and the forest root. Also, SSSD by default tries to resolve all groups the user is a member of, from all domains. In case the SSSD client is behind a firewall preventing connection to a trusted domain, can set the ``ad_enabled_domains`` option to selectively enable only the reachable domains.

* SSSD keeps switching to offline mode with a ``DEBUG`` message saying ``Service resolving timeout reached``

  * This might happen if the service resolution reaches the configured time out before SSSD is able to perform all the steps needed for service resolution in a complex AD forest, such as locating the site or cycling over unreachable DCs. Please check the ``FAILOVER`` section in the man pages Often, increasing the ``dns_resolver_timeout`` option helps to allow more time for the service discovery.

* A group my user is a member of doesn't display in the ``id`` output

  * Cases like this are best debugged from an empty cache. Check if the group GID appears in the output of ``id -G`` first. In case the group is not present in the ``id -G`` output at all, there is something up with the initgroups part. Check the schema and look for anything strange during the initgr operation in SSSD back end logs. If the group is present in ``id -G`` output but not in ``id`` output (or a subsequent id output) then there's something wrong with resolving the group GIDs with ``getgrgid()``.

* Only the primary group my user is assigned is displayed in the ``id`` output

  * This is commonly caused by removing ``Authenticated Users`` from the ``Pre-Windows 2000 Compatible Access`` group to harden Active Directory. 

  * The computer account created when joining the domain (normally ``COMPUTERNAME$``) needs to be able to access the ``memberOf`` attribute of the user you're performing an ``id`` against. Removing ``Authenticated Users`` from the ``Pre-Windows 2000 Compatible Access`` removes that ability.

  * To grant the required permission, you can do any of the following (in order of least preferable  to most preferable)

    * Re-add ``Authenticated Users`` to the ``Pre-Windows 2000 Compatible Access``

    * Add the ``DOMAIN\Domain Computers`` account to the ``Pre-Windows 2000 Compatible Access``. This would result in a widely scoped ACL (in a default Active Directory configuration) where the ``DOMAIN\Domain Computers`` group has permission to read all attributes on all User, Group and InetOrgPerson objects in the domain. Bearing in mind you've likely run into problems because ``Authenticated Users`` has been removed from the ``Pre-Windows 2000 Compatible Access`` group this is probably undesirable.

    * Add the ``COMPUTERNAME$`` account to the ``Pre-Windows 2000 Compatible Access``. This would result in a widely scoped ACL (in a default Active Directory configuration) where the ``COMPUTERNAME$`` account has permission to read all attributes on all User, Group and InetOrgPerson objects in the domain. Bearing in mind you've likely run into problems because ``Authenticated Users`` has been removed from the ``Pre-Windows 2000 Compatible Access`` group this is probably undesirable.

    * Grant the ``Read remote access information`` permission to the OU the user(s) are in for the ``COMPUTERNAME$`` account. This would result in a narrowly scoped ACL where the ``COMPUTERNAME$`` account has permission to read just the attributes required but would be add an administrative overhead of having to add this permission for each Linux ``COMPUTERNAME$``. Would quickly create a messy ACL on the OU.

    * Grant the ``Read remote access information`` permission on the OU the user(s) are in for the ``DOMAIN\Domain Computers``. This would result in a more broadly scoped ACL where all computers in the domain have permission to read just the attributes required, but without the administrative overhead of having to add this permission for each Linux computer. Note, this would grant **all windows computers** the same permission which doesn't appear to be required (I'm not sure how windows computers perform group enumeration, but it works when ``Authenticated Users`` has been removed from the ``Pre-Windows 2000 Compatible Access`` group) and would therefore be against the *principle of least privilege*.

    * Create a group called (for example) ``Linux Computers`` and then grant the ``Read remote access information`` permission on the OU the user(s) are in for the ``DOMAIN\Linux Computers`` group and add all your Linux ``COMPUTERNAME$`` accounts to the ``DOMAIN\Linux Computers`` group. This would result in a narrowest scoped ACL where members of the ``DOMAIN\Linux Computers`` group have permission to read just the attributes required with a small administrative overhead of having to add the Linux ``COMPUTERNAME$`` account to the ``DOMAIN\Linux Computers`` group as new machines are added.

  * If you wish to implement one of the solutions where you do not alter the membership of the ``Pre-Windows 2000 Compatible Access`` group, you can do the following...

    * Open **Active Directory Users and Groups**

    * Navigate to EITHER 

      * The root of your domain (NOT RECOMMENDED if following a principle of least privilege approach. Think about your highly privileged accounts, e.g. Domain Admins etc)

      * The OU where your users are located (by default that'll be domain -> Users, but in a mature domain this is likely to be elsewhere. Ideally your highly privileged accounts will be in a different OU)

    * Right click on the domain root / OU, click the **Properties**, and select the **Security** tab.

    * Click the **Advanced** button

    * Click **Add**

    * Click **Select a principal**. Set this as either  ``DOMAIN\Domain Computers``, ``DOMAIN\Linux Computers`` or ``COMPUTERNAME$`` depending on your chosen administrative strategy.

    * Select **Type** as ``Allow``

    * Select **Applies To** as ``Descendant User objects``

    * Select ``Read remote access information`` and make sure **all** other boxes are **unticked**.

  * I found the GUI would by default tick all the Read permissions. To save me having to untick everything by hand I discovered that making sure all the boxes in the Permissions section were Unticked, then switching **Applies to** from ``Descendant User objects`` to ``Descendant Trusted Domain objects`` and back to ``Descendant User objects`` would untick everything. Then I could just tick ``Read remote access information`` :)

  * A thread discussing this topic is available on sssd-users mailing list at https://lists.fedorahosted.org/archives/list/sssd-users@lists.fedorahosted.org/thread/GR2YGPZKOTDFTMSC34J36BZY2N5IQ7M3/
