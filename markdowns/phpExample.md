## PHP - Entity, Service, Controller, And Route Example

This is an example of a typical entity, service, controller, route stack I added to our main PHP API.  I have done several of these, but this one was the most recent.

---


#### This is the entity itself.  Here you can see our ORM, Doctrine, at play.
```
<?php

namespace Skyetel\Entities;

/**
 * @Entity
 * @Table(name="stonehenge_vlb")
 **/

class StonehengeVLB implements \JsonSerializable, Patchable 
{

    public static $INCLUDABLE_FIELDS = [
        "id",
        "name",
        "ip_private",
        "ip_public",
        "notes",
        "created_by",
        "created_on",
        "updated_by",
        "updated_on",
        "archived",
        "allocated_cps"
    ];

     /**
     * @Id @GeneratedValue @Column(type="integer", options={"unsigned":true})
     */
    private $id;

     /**
     * @Column(type="string", length=255, nullable=false)
     */
    private $name;

    /**
     * @Column(type="string", length=255, nullable=false)
     */
    private $ip_private;

    /**
     * @Column(type="string", length=255, nullable=false)
     */
    private $ip_public;

    /**
     * @Column(type="integer", nullable=true)
     */
    private $allocated_cps;

    /**
     * @Column(type="text", nullable=true)
     */
    private $notes;

    /**
     * @Column(type="string", length=255, nullable=false)
     */
    private $created_by;

    /**
     * @Column(type="string", length=255, nullable=false)
     */
    private $updated_by;

    /**
     * @Column(type="datetime")
     */
    private $created_on;

    /**
     * @Column(type="datetime")
     */
    private $updated_on;

    /**
     * @Column(type="boolean")
     */
    private $archived;

    /**
     * @ManyToOne(targetEntity="StonehengeVendorTermTrunkGroups", inversedBy="stonehenge_vlb")
     */
    private $termination_vendor_trunkgroup;

    // Constructor
    function __construct()
    {
        $this->created_on = new \DateTime;
    }



    //Getters and Setters
    public function getId()
    {
        return $this->id;
    }

    public function setId($id) 
    {
        $this->id = $id;
    }

    public function getName()
    {
        return $this->name;
    }

    public function setName($name) 
    {
        $this->name = $name;
    }

    public function getIpPrivate()
    {
        return $this->ip_private;
    }

    public function setIpPrivate($ip_private) 
    {
        $this->ip_private = $ip_private;
    }

    public function getIpPublic()
    {
        return $this->ip_public;
    }

    public function setIpPublic($ip_public) 
    {
        $this->ip_public = $ip_public;
    }

    public function getNotes()
    {
        return $this->notes;
    }

    public function setNotes($notes) 
    {
        $this->notes = $notes;
    }

    public function getAllocatedCps()
    {
        return $this->allocated_cps;
    }

    public function setAllocatedCps($allocated_cps) 
    {
        $this->allocated_cps = $allocated_cps;
    }

    public function getCreatedOn()
    {
        return $this->created_on;
    }

    public function setCreatedOn($created_on)
    {
        $this->created_on = $created_on;
    }

    public function getUpdatedOn()
    {
        return $this->updated_on;
    }

    public function setUpdatedOn($updated_on)
    {
        $this->updated_on = $updated_on;
    }

    public function getUpdatedBy()
    {
        return $this->updated_by;
    }

    public function setUpdatedBy($updated_by)
    {
        $this->updated_by = $updated_by;
    }

    public function getCreatedBy()
    {
        return $this->created_by;
    }

    public function setCreatedBy($created_by)
    {
        $this->created_by = $created_by;
    }

    public function getArchived() {
        return $this->archived;
    }
    
    function setArchived($archived) {
        $this->archived = $archived;
    }



    // Patch
    function patch(array $patch)
    {
        $this->name = !array_key_exists('name', $patch) ?
            $this->name : $patch['name'];
        $this->ip_private = !array_key_exists('ip_private', $patch) ?
            $this->ip_private : $patch['ip_private'];
        $this->ip_public = !array_key_exists('ip_public', $patch) ?
            $this->ip_public : $patch['ip_public'];
        $this->updated_by = !array_key_exists('updated_by', $patch) ?
            $this->updated_by : $patch['updated_by'];
        $this->notes = !array_key_exists('notes', $patch) ?
            $this->notes : $patch['notes'];
        $this->updated_on = new \DateTime;
        $this->archived = !array_key_exists('archived', $patch) ?
            $this->archived : $patch['archived']; 
    }

    // JsonSerializable
    public function jsonSerialize()
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'ip_private' => $this->ip_private,
            'ip_public' => $this->ip_public,
            'notes' => $this->notes,
            'created_by' => $this->created_by,
            'updated_by' => $this->updated_by,
            'created_on' => $this->created_on,
            'updated_on' => $this->updated_on,
            'archived' => $this->archived,
            'allocated_cps' => $this->allocated_cps,
        ];
    }

}
```


---


#### This is the service file.
```
<?php 

namespace Skyetel\Services\StonehengeTrunkGroups;

use Respect\Validation\Validator as v;
use Skyetel\Services\Mapping\IntegerField;
use Skyetel\Services\Mapping\StringField;
use Skyetel\Services\Mapping\DateField;
use Skyetel\Services\Mapping\BooleanField;
use Respect\Validation\Exceptions\ValidationException;
use Skyetel\Services\Traits\GetItemListTrait;
use Doctrine\ORM\Query\ResultSetMappingBuilder;

class StonehengeVLBService
{
    private $log;
    private $database;

    use GetItemListTrait;

    public function __construct($database, $log)
    {
        $this->log = $log;
        $this->database = $database;
    }

    public function getMappingFields()
    {
        v::with('Skyetel\\Validation\\Rules\\');

        return [
            'id' => (new IntegerField('sv.id'))
                ->setValidation(v::intVal()->min(1)->setName('id')),
            'name' => (new StringField('sv.name'))
                ->setValidation(v::stringType()->length(1, 255)->setName('name')),
            'ip_private' => (new StringField('sv.ip_private'))
                ->setValidation(v::stringType()->length(1, 255)->setName('ip_private')),
            'ip_public' => (new StringField('sv.ip_public'))
                ->setValidation(v::stringType()->length(1, 255)->setName('ip_public')),
            'notes' => (new StringField('sv.notes')),
            'created_by' => (new StringField('sv.created_by'))
                ->setValidation(v::stringType()->length(1, 255)->setName('created_by')),
            'updated_by' => (new StringField('sv.updated_by'))
                ->setValidation(v::stringType()->length(1, 255)->setName('updated_by')),
            'created_on' => (new DateField('sv.created_on')),   
            'updated_on' => (new DateField('sv.updated_on')), 
            'archived' => (new BooleanField('sv.archived')),   
        ];
    }

    public function getItemList($queryParameters, $cache = null)
    {
        $filters = $queryParameters->getFilters();
        $parameters = $queryParameters->getFilterParameters();
        $sorting = $queryParameters->getSortFields();
        $rsm = new ResultSetMappingBuilder($this->database);
        $rsm->addRootEntityFromClassMetadata('Skyetel\Entities\StonehengeVLB', 'sv');


        $filters =  count($filters) > 0 ? " WHERE sv.archived=0 AND " . implode(" AND ", $filters) : ' WHERE sv.archived=0 ';
        $sorting =  count($sorting) > 0 ? " ORDER BY " . implode(",", $sorting) : '';

        // please note that the coalesce section of this query was added later by another developer
        $sql = "SELECT sv.id, sv.name, sv.ip_private, sv.ip_public, sv.created_by, sv.created_on, sv.updated_by, sv.updated_on, sv.notes, sv.archived, COALESCE(sum(tg.cps_limit),0) as allocated_cps FROM stonehenge_vlb sv  LEFT JOIN stonehenge_termination_vendor_trunkgroup tg on sv.id=tg.vlb_id  group by sv.id order by sv.id";

        $list = $this->database->createNativeQuery($sql, $rsm);

        foreach ($parameters as $key => $value) {
            $list->setParameter($key, $value);
        }

        $list
            ->useResultCache(true, 60)
            ->setParameter("limit", $queryParameters->getLimit())
            ->setParameter("offset", $queryParameters->getOffset());

        return $list->getResult();
    }

    public function createItem($post, $user_email)
    {
        
        $item = new \Skyetel\Entities\StonehengeVLB;

        $item->setCreatedBy($user_email);

        if (empty($post['name'])) {
            throw new ValidationException('Provided Vendor Name not valid');
        } else {
            $item->setName($post['name']);
        }

        if (empty($post['ip_private'])) {
            throw new ValidationException('Provided Private IP not valid');
        } else {
            $item->setIpPrivate($post['ip_private']);
        }

        if (empty($post['ip_public'])) {
            throw new ValidationException('Provided Public IP not valid');
        } else {
            $item->setIpPublic($post['ip_public']);
        }
            
        if (!empty($post['notes'])) {
            $item->setNotes($post['notes']);
        }

        $item->setArchived(false);
        $this->database->persist($item);
        $this->database->flush();

        return $item;
    }

    public function getItem($id)
    {
        return $this->database
            ->getRepository('Skyetel\Entities\StonehengeVLB')
            ->findOneBy(["id" => $id]);
    }


    public function updateItem($item, $post, $user_email)
    {

        if (!empty($item)) {
            $vlb_entry_check = $this->database
                ->getRepository('Skyetel\Entities\StonehengeVLB')
                ->findOneBy(['id' => $item]);

            if (empty($vlb_entry_check)) {
                throw new ValidationException('Provided VLB ID not valid');
            }


        } else {
            throw new ValidationException('Provided VLB ID missing');
        }

        $post['updated_by']=$user_email;

        $item->patch($post);
        $this->database->flush();

        return $item;
    }


    public function deleteItem($item, $user_email)
    {
        if (!empty($item)) {
            $vlb_entry_check = $this->database
                ->getRepository('Skyetel\Entities\StonehengeVLB')
                ->findOneBy(['id' => $item]);

            if (empty($vlb_entry_check)) {
                throw new ValidationException('Provided VLB ID not valid');
            }

        } else {
            throw new ValidationException('Provided VLB ID not found');
        }

        $post['updated_by']=$user_email;
        $post['archived']=true;

        $item->patch($post);
        $this->database->flush();
    }
}
```

---

#### And the controller which has an instance of Doctrine's "Entity Manager" that I passed to the service.

```
public function getStonehengeVendorLoadBalancers(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
    {
        $service = new StonehengeVLBService($this->read_connection, $this->log);
        $user = RequestHelper::getUser($request);
        $queryParams = new QueryParameters($request, $service);

        $result = $service->getItemList($queryParams);

        return Response::success($response, $result);
    }

    public function getStonehengeVendorLoadBalancer(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
    {
        $service = new StonehengeVLBService($this->read_connection, $this->log);
        $id = $service->getItem($request->getAttribute('id'));

        try {
            $object = $service->getItem($id);
        } catch (SkyetelFailException $e) {
            return Response::error($response, $e->getMessage(), $e->getCode());
        }

        return Response::success($response, $object);
    }

    public function createStonehengeVendorLoadBalancer(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
    {
        $service = new StonehengeVLBService($this->write_connection, $this->log);

        $user = RequestHelper::getUser($request);
        $user_mail = $user->getMail();

        $allowed_fields = array_keys($service->getMappingFields());
        $data = RequestHelper::getAllowedRequestFields($request, $allowed_fields);

        Validation::validateRules($data, $service->getMappingFields());

        try {
            $object = $service->createItem($data, $user_mail);
        } catch (SkyetelFailException $e) {
            return Response::error($response, $e->getMessage(), $e->getCode());
        }

        return Response::success($response, $object);
    }

    public function updateStonehengeVendorLoadBalancer(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
    {
        $service = new StonehengeVLBService($this->write_connection, $this->log);
        $id = $service->getItem($request->getAttribute('id'));
        $allowed_fields = array_keys($service->getMappingFields());
        $data = RequestHelper::getAllowedRequestFields($request, $allowed_fields);
        $user = RequestHelper::getUser($request);
        $user_mail = $user->getMail();

        Validation::validateRules($data, $service->getMappingFields());

        try {
            $object = $service->updateItem($id, $data, $user_mail);
        } catch (SkyetelFailException $e) {
            return Response::error($response, $e->getMessage(), $e->getCode());
        }

        return Response::success($response, $object);
    }

    public function deleteStonehengeVendorLoadBalancer(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
    {
        $service = new StonehengeVLBService($this->write_connection, $this->log);
        $user = RequestHelper::getUser($request);
        $user_mail = $user->getMail();

        $id = $service->getItem($request->getAttribute('id'));
        $service->deleteItem($id, $user_mail);

        return Response::success($response, ["SUCCESS" => "Stonehenge Vendor Load Balancer Deleted"]);
    }
```

---

####  And, of course, the SLIM routes
```
$group->get('/vendorloadbalancers' StonehengeTrunkgroupController::class . ':getStonehengeVendorLoadBalancers');
$group->post('/vendorloadbalancers', StonehengeTrunkgroupController::class . ':createStonehengeVendorLoadBalancer');
$group->get('/vendorloadbalancers/{id}', StonehengeTrunkgroupController::class . ':getStonehengeVendorLoadBalancer');
$group->patch('/vendorloadbalancers/{id}', StonehengeTrunkgroupController::class . ':updateStonehengeVendorLoadBalancer');
$group->delete('/vendorloadbalancers/{id}', StonehengeTrunkgroupController::class . ':deleteStonehengeVendorLoadBalancer');
```

[Return to main menu](../README.md)