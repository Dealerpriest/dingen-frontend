<template>
  <div class="browser">
    <v-snackbar v-model="showInfo" bottom right :timeout="3000" color="green">
      {{ infoText }}
      <v-btn dark flat @click="showInfo = false">
        <v-icon>close</v-icon>
      </v-btn>
    </v-snackbar>
    <v-snackbar v-model="showActionSnackbar" :timeout="0" top color="grey">
      <v-tooltip bottom open-delay="1000">
        <template v-slot:activator="{ on }">
          <v-btn v-on="on" dark flat @click="moveSelectedObjects">
            <v-icon>mdi-folder-move</v-icon>
          </v-btn>
        </template>
        <span>Flytta markerade objekt</span>
      </v-tooltip>
      <v-tooltip bottom open-delay="1000">
        <template v-slot:activator="{ on }">
          <v-btn v-on="on" dark flat @click="duplicateSelectedObj">
            <v-icon>mdi-content-duplicate</v-icon>
          </v-btn>
        </template>
        <span>Duplicera markerade objekt</span>
      </v-tooltip>
      <v-tooltip bottom open-delay="1000">
        <template v-slot:activator="{ on }">
          <v-btn v-on="on" dark flat @click="convertSelectedObjects">
            <v-icon>mdi-autorenew</v-icon>
          </v-btn>
        </template>
        <span>Konvertera markerade objekt</span>
      </v-tooltip>
      <v-tooltip bottom open-delay="1000">
        <template v-slot:activator="{ on }">
          <v-btn v-on="on" dark flat @click="deleteSelectedObjects">
            <v-icon>delete</v-icon>
          </v-btn>
        </template>
        <span>Tabort markerade objekt.</span>
      </v-tooltip>
      <v-tooltip bottom open-delay="1000">
        <template v-slot:activator="{ on }">
          <v-btn v-on="on" dark flat @click="unselectAll">
            <v-icon>mdi-selection-off</v-icon>
          </v-btn>
        </template>
        <span>Avmarkerade alla objekt.</span>
      </v-tooltip>
    </v-snackbar>
    <v-dialog v-model="errorDialog" max-width="300px">
      <v-card>
        <v-card-title class="headline" color="red">OOpss..!</v-card-title>
        <v-card-text>{{errorMsg}}</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="gray darken-1" flat @click="errorDialog = false">OK</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-dialog v-model="warningDialog" max-width="300px">
      <v-card>
        <v-card-title class="headline">Varning.</v-card-title>
        <v-card-text>{{warningMsg}}</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="gray darken-1" flat @click="selectedObjectAction()">OK</v-btn>
          <v-btn
            color="gray darken-1"
            flat
            @click="warningDialog = false; selectedObjectAction = ()=>{}"
          >Avrbyt</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-dialog v-model="moveDialog" max-width="700px">
      <v-card>
        <v-card-title class="headline">Flytta {{selectedObject? selectedObject.obj.get("name"): ""}}</v-card-title>
        <BrowseContainer ref="browseContainer" @conChanged="moveConChanged" />
        <v-layout flex width="100%" row justify-center style="padding-bottom: 16px">
          <v-btn large @click="selectedObjectAction">
            <v-icon class="mr-2">mdi-arrow-expand-right</v-icon>flytta
          </v-btn>
        </v-layout>
      </v-card>
    </v-dialog>
    <div class="wrapper-view">
      <div class="main-view">
        <v-progress-linear
          indeterminate
          :active="loadingDialog"
          height="5"
          style="margin: 0; position: fixed; top: 48px"
        ></v-progress-linear>
        <div class="breadcrumb">
          <v-btn large icon @click="goToParent">
            <v-icon large>arrow_back</v-icon>
          </v-btn>
          <v-breadcrumbs :items="breadcrumbs" divider=">">
            <template v-slot:item="props">
              <a
                v-if="!props.item.disabled"
                @click="$router.push('/browser/' + props.item.id)"
              >{{ props.item.text }}</a>
              <p v-else style="margin: 0">
                <strong>{{ props.item.text }}</strong>
              </p>
            </template>
          </v-breadcrumbs>
        </div>
        <div>
          <v-layout
            flex
            row
            align-center
            style="width: 100%; padding: 0 30px 20px 30px; justify-content: space-between;"
          >
            <p class="title font-weight-light" style="margin: 0">Containers:</p>
            <v-btn-toggle v-model="containerView" mandatory>
              <v-tooltip bottom open-delay="1000">
                <template v-slot:activator="{ on }">
                  <v-btn v-on="on" flat>
                    <v-icon>mdi-view-grid</v-icon>
                  </v-btn>
                </template>
                <span>Grid vy</span>
              </v-tooltip>
              <v-tooltip bottom open-delay="1000">
                <template v-slot:activator="{ on }">
                  <v-btn v-on="on" flat>
                    <v-icon>mdi-view-sequential</v-icon>
                  </v-btn>
                </template>
                <span>List vy</span>
              </v-tooltip>
            </v-btn-toggle>
          </v-layout>
          <ObjectView
            :objects="containers"
            :view="['block','list'][containerView]"
            :toc="false"
            :selected.sync="containersSelected"
            @objSelected="objSelected"
            @openCrForm="openCrForm"
            @openObj="openObject"
            @moveObj="setMovableObj"
            @dupObj="setDuplicateObj"
            @convObj="setConvertObj"
            @deleteObj="setDeletableObj"
            @info="openInfoView"
          ></ObjectView>
          <v-layout
            flex
            row
            align-center
            style="width: 100%; padding: 0 30px 20px 30px; justify-content: space-between;"
          >
            <p class="title font-weight-light" style="margin: 0">Saker:</p>
            <v-btn-toggle v-model="thingsView" mandatory>
              <v-tooltip bottom open-delay="1000">
                <template v-slot:activator="{ on }">
                  <v-btn v-on="on" flat>
                    <v-icon>mdi-view-grid</v-icon>
                  </v-btn>
                </template>
                <span>Grid vy</span>
              </v-tooltip>
              <v-tooltip bottom open-delay="1000">
                <template v-slot:activator="{ on }">
                  <v-btn v-on="on" flat>
                    <v-icon>mdi-view-sequential</v-icon>
                  </v-btn>
                </template>
                <span>List vy</span>
              </v-tooltip>
            </v-btn-toggle>
          </v-layout>
          <ObjectView
            :objects="things"
            :view="['block','list'][thingsView]"
            :toc="true"
            :selected.sync="thingsSelected"
            @objSelected="objSelected"
            @openCrForm="openCrForm"
            @openObj="openObject"
            @moveObj="setMovableObj"
            @dupObj="setDuplicateObj"
            @convObj="setConvertObj"
            @deleteObj="setDeletableObj"
            @info="openInfoView"
          ></ObjectView>
        </div>
      </div>
      <v-divider vertical></v-divider>
      <div v-show="createDialog" class="side-view">
        <div style="position: relative; z-index: 1;">
          <v-btn
            color="normal"
            @click="createDialog = false"
            style="position: absolute; top: 0; right: 0;"
            icon
          >
            <v-icon>close</v-icon>
          </v-btn>
        </div>

        <CreateObject
          ref="crudForm"
          :curCon="currentObject"
          :showBrowser="false"
          @updOrCr="updOrCrEvent"
          @close="closeCreateDialog"
        />
      </div>

      <div v-if="detailedView && selectedObject != null" class="side-view">
        <div style="position: relative; z-index: 1;">
          <v-btn
            color="normal"
            @click="detailedView = false"
            style="position: absolute; top: 0; right: 0;"
            icon
          >
            <v-icon>close</v-icon>
          </v-btn>
        </div>
        <ObjectInfoView
          :edit="true"
          :selectedObject="selectedObject"
          @editBtn="openCrForm({obj: selectedObject.obj, item: selectedObject.obj.className == 'Thing'})"
          @deleteBtn="setDeletableObj(selectedObject.obj)"
        />
      </div>
      <!-- <v-footer class="mt-5">Footer</v-footer> -->
    </div>
  </div>
</template>

<script lang="ts">
import Vue from "vue";
import Component from "vue-class-component";
import marked from "marked";
import Parse from "parse";
import CreateObject from "@/components/CreateObject.vue";
import BrowseContainer from "@/components/BrowseContainer.vue";
import ObjectView from "@/components/ObjectView.vue";
import ObjectInfoView from "@/components/ObjectInfoView.vue";

@Component({
  // @ts-ignore
  components: {
    CreateObject,
    BrowseContainer,
    ObjectView,
    ObjectInfoView
  }
})
export default class Browser extends Vue {
  containers: any[] = [];
  containersSelected: any[] = [];
  things: any[] = [];
  thingsSelected: any[] = [];
  currentObject: any = null;
  selectedObject: any = null;
  selectedObjectAction: any = () => {};
  thingInformation: any = {};
  breadcrumbs: any[] = [];
  createDialog: boolean = false;
  moveDialog: boolean = false;
  containerSelectedToMove: any = null;
  errorMsg: string = "";
  errorDialog: boolean = false;
  loadingDialog: boolean = false;
  warningDialog: boolean = false;
  warningMsg: string = "";
  showInfo: boolean = false;
  infoText: string = "";

  containerView: any = 0;
  thingsView: any = 1;
  detailedView: boolean = false;

  beforeRouteEnter(to: any, from: any, next: any) {
    console.log("beforeRouteEnter triggered");
    next((vm: any) => {
      if (to.params.id) {
        vm.openObjectById(to.params.id);
      } else {
        vm.loadingDialog = true;
        vm.changeContainer(null);
      }
    });
  }

  beforeRouteUpdate(to: any, from: any, next: any) {
    console.log("beforeRouteUpdate triggered");
    if (to.params.id) {
      this.openObjectById(to.params.id);
    } else {
      this.loadingDialog = true;
      this.changeContainer(null);
    }
    next();
  }

  beforeRouteLeave(to: any, from: any, next: any) {
    console.log("beforeRouteLeave triggered");
    next();
  }

  goToParent() {
    if (this.currentObject) {
      this.loadingDialog = true;
      const parent = this.currentObject.get("parent");
      if (parent) {
        this.$router.push("/browser/" + parent.id);
      } else {
        this.$router.push("/browser");
      }
    }
  }

  beforeMount() {
    console.log("beforeMount triggered");
    // if (this.$route.params.id) {
    //     this.openObjectById(this.$route.params.id);
    // } else {
    //     this.loadingDialog = true;
    //     this.changeContainer(null);
    // }
  }

  error(msg: any) {
    this.errorMsg = msg.toString();
    this.errorDialog = true;
  }

  info(msg: any) {
    this.infoText = msg.toString();
    this.showInfo = true;
  }

  markedDesc(obj: any) {
    return marked(obj.get("description"), {
      sanitize: true
    });
  }

  formatDateTime(date: Date) {
    return date.toLocaleString();
  }

  async closeCreateDialog(editObj: any) {
    console.log(editObj);
    if (editObj != null) {
      if (editObj.className == "Container") {
        this.objSelected(await this.$store.dispatch("parseContainer", editObj));
      } else {
        this.objSelected(await this.$store.dispatch("parseThing", editObj));
      }

      this.createDialog = false;
      this.detailedView = true;
    } else {
      this.createDialog = false;
    }
  }

  async updateThingInfo() {
    this.thingInformation = {
      amount: this.currentObject.get("amount"),
      name: this.currentObject.get("name"),
      description: this.markedDesc(this.currentObject),
      tags: await this.currentObject
        .get("tags")
        .query()
        .find()
    };
  }

  async openObjectById(id: string) {
    this.loadingDialog = true;
    if (id) {
      const query = new Parse.Query("Container");
      query.equalTo("objectId", id);
      let result = await query.first();

      if (!result) {
        const thingQuery = new Parse.Query("Thing");
        thingQuery.equalTo("objectId", id);
        result = await thingQuery.first();
      }

      if (result!.className == "Container") {
        this.changeContainer(result);
      } else {
        this.$store
          .dispatch("getBreadCrumbs", this.currentObject)
          .then((result: any) => {
            this.breadcrumbs = result;
          });
        this.objSelected(await this.$store.dispatch("parseThing", result));
      }
    } else {
      this.changeContainer(null);
    }
  }

  openObject(obj: any) {
    console.log("opening new object!!");
    this.loadingDialog = true;

    if (obj) {
      this.$router.push("/browser/" + obj.obj.id);
    } else {
      this.$router.push("/browser/");
    }
  }

  unselectAll() {
    this.thingsSelected = [];
    this.containersSelected = [];
  }

  objSelected(obj: any) {
    if (obj) {
      this.selectedObject = obj;
      if (obj.obj.className == "Container") {
        this.thingsSelected = [];
      } else {
        this.containersSelected = [];
      }
    } else {
      this.selectedObject = null;
      this.unselectAll();
    }
  }

  get selectedObjects() {
    return [
      ...this.thingsSelected.map(x => x.obj),
      ...this.containersSelected.map(x => x.obj)
    ];
  }

  get showActionSnackbar() {
    return this.selectedObjects.length > 1;
  }

  openInfoView(obj: any) {
    this.createDialog = false;
    this.detailedView = true;
  }

  async changeContainer(container: any) {
    console.log("change container triggered");
    if (this.currentObject != container) {
      this.unselectAll();
    }
    this.currentObject = container;

    const loadContainers = async () => {
      let query: any = null;

      if (this.currentObject === undefined || this.currentObject === null) {
        const nullQuery = new Parse.Query("Container");
        nullQuery.equalTo("parent", null);

        const undefinedQuery = new Parse.Query("Container");
        undefinedQuery.equalTo("parent", undefined);

        query = Parse.Query.or(nullQuery, undefinedQuery);
      } else {
        query = new Parse.Query("Container");
        query.equalTo("parent", this.currentObject);
      }
      let results = await query.find();

      if (results) {
        this.containers = [];
        return Promise.all(
          results.map(async (con: any) => {
            this.containers.push(
              await this.$store.dispatch("parseContainer", con)
            );
          })
        );
      } else {
        this.containers = [];
      }
    };

    Promise.all([
      loadContainers(),
      this.$store.dispatch("getBreadCrumbs", this.currentObject),
      this.updateThingsArray()
    ])
      .then((results: any[]) => {
        this.breadcrumbs = results[1];
      })
      .finally(() => {
        this.loadingDialog = false;
      });
  }

  async updateThingsArray() {
    let query: any = null;
    if (this.currentObject === undefined || this.currentObject === null) {
      const nullQuery = new Parse.Query("Thing");
      nullQuery.equalTo("parent", null);

      const undefinedQuery = new Parse.Query("Thing");
      undefinedQuery.equalTo("parent", undefined);

      query = Parse.Query.or(nullQuery, undefinedQuery);
    } else {
      query = new Parse.Query("Thing");
      query.equalTo("parent", this.currentObject);
    }

    this.things = [];
    const things = await query.find();
    if (things) {
      return Promise.all(
        things.map(async (thing: any) => {
          this.things.push(await this.$store.dispatch("parseThing", thing));
        })
      );
    }
  }

  public openCrForm(params: any) {
    this.createDialog = true;
    this.detailedView = false;
    const createComponent = <CreateObject>this.$refs.crudForm;
    createComponent.setFormData(params.obj, params.item);
  }

  setMovableObj(obj: any) {
    const form = <BrowseContainer>this.$refs.browseContainer;
    form.changeContainer(this.currentObject);
    this.moveDialog = true;
    this.selectedObjectAction = () => {
      this.loadingDialog = true;
      this.$store
        .dispatch("moveObject", {
          obj: this.selectedObject.obj,
          selectedContainer: this.containerSelectedToMove
        })
        .then((result: any) => {
          this.selectedObjectAction = () => {};
          this.changeContainer(this.currentObject);
          this.info(
            "Flyttade " +
              result.get("name") +
              " till " +
              (this.containerSelectedToMove
                ? this.containerSelectedToMove.get("name")
                : "Start")
          );
          this.moveDialog = false;
        })
        .catch((err: any) => {
          this.error(
            "Kunde inte flytta " +
              obj.get("name") +
              " till " +
              (this.containerSelectedToMove
                ? this.containerSelectedToMove.get("name")
                : "Start") +
              ". \n" +
              err
          );
        })
        .finally(() => {
          this.loadingDialog = false;
        });
    };
  }

  moveSelectedObjects() {
    const form = <BrowseContainer>this.$refs.browseContainer;
    form.changeContainer(this.currentObject);
    this.moveDialog = true;
    this.selectedObjectAction = () => {
      let promises = this.selectedObjects.map(x =>
        this.$store.dispatch("moveObject", {
          obj: x,
          selectedContainer: this.containerSelectedToMove
        })
      );
      Promise.all(promises)
        .then(results => {
          this.selectedObjectAction = () => {};
          this.info(
            "Flyttade " +
              results.length +
              " objekt till " +
              (this.containerSelectedToMove
                ? this.containerSelectedToMove.get("name")
                : "Start")
          );
        })
        .catch(err => {
          this.error(
            "Kunde inte flytta object till " +
              (this.containerSelectedToMove
                ? this.containerSelectedToMove.get("name")
                : "Start") +
              ". \n" +
              err
          );
        })
        .finally(() => {
          this.unselectAll();
          this.changeContainer(this.currentObject);
        });
    };
  }

  moveConChanged(con: any) {
    this.containerSelectedToMove = con;
  }

  setConvertObj(obj: any) {
    this.warningDialog = true;
    this.warningMsg =
      "Vill du konvertera '" +
      obj.get("name") +
      "' till en " +
      (obj.className == "Thing" ? "Container?" : "Sak?");
    this.selectedObjectAction = () => {
      this.$store
        .dispatch("convertObject", this.selectedObject.obj)
        .then(async (result: any) => {
          if (result.className == "Container") {
            this.info(
              "Konverterade '" + result.get("name") + "' till en Container"
            );
            this.things.splice(
              this.things.findIndex(x => x.obj.id == obj.id),
              1
            );
            this.updOrCrEvent(
              await this.$store.dispatch("parseContainer", result),
              false
            );
          } else {
            this.info("Konverterade '" + result.get("name") + "' till en Sak");
            //this.updOrCrEvent(await this.$store.dispatch("parseThing", result), false)
            this.changeContainer(this.currentObject);
          }
        })
        .catch(error => {
          this.error("Fel uppstod vi konvertering. \n" + error);
          console.error(error);
        })
        .finally(() => {
          this.warningDialog = false;
          this.selectedObjectAction = () => {};
        });
    };
  }

  convertSelectedObjects() {
    this.warningDialog = true;
    this.warningMsg =
      "Vill du konvertera " + this.selectedObjects.length + " objekt?";
    this.selectedObjectAction = () => {
      this.warningDialog = false;
      this.loadingDialog = true;
      let promises = this.selectedObjects.map(x =>
        this.$store.dispatch("convertObject", x)
      );
      Promise.all(promises)
        .then(results => {
          this.info("Konverterade " + results.length + " saker!");
        })
        .catch(error => {
          this.error("Fel uppstod vi konvertering. \n" + error);
          console.error(error);
        })
        .finally(() => {
          this.changeContainer(this.currentObject);
          this.unselectAll();
          this.selectedObjectAction = () => {};
        });
    };
  }

  setDuplicateObj(obj: any) {
    this.warningDialog = true;
    this.selectedObjectAction = () => {
      this.warningDialog = false;
      this.$store
        .dispatch("duplicateObject", this.selectedObject.obj)
        .then(async (result: any) => {
          this.info("Duplicerade " + result.get("name"));
          if (result.className == "Container") {
            this.updOrCrEvent(
              await this.$store.dispatch("parseContainer", result),
              false
            );
          } else {
            this.updOrCrEvent(
              await this.$store.dispatch("parseThing", result),
              false
            );
          }
        })
        .catch(err => {
          console.error(err);
          this.error("Kunde inte duplicera. " + err);
        })
        .finally(() => {
          this.selectedObjectAction = () => {};
        });
    };
    this.warningMsg = "Vill du duplicera '" + obj.get("name") + "'?";
  }

  duplicateSelectedObj() {
    this.warningDialog = true;
    this.warningMsg =
      "Är du säker på att du vill duplicera " +
      this.selectedObjects.length +
      " object?";
    this.selectedObjectAction = () => {
      this.warningDialog = false;
      let promises = this.selectedObjects.map(x =>
        this.$store.dispatch("duplicateObject", x)
      );
      Promise.all(promises)
        .then(results => {
          this.info("Duplicerade " + results.length + " objekt ");
        })
        .catch(err => {
          this.error("Kunde inte duplicera objecten. \n" + err);
        })
        .finally(() => {
          this.selectedObjectAction = () => {};
          this.changeContainer(this.currentObject);
        });
    };
  }

  setDeletableObj(obj: any) {
    this.warningDialog = true;
    this.selectedObjectAction = () => {
      this.warningDialog = false;
      this.loadingDialog = true;
      this.$store
        .dispatch("deleteObject", obj)
        .then((result: any) => {
          this.info("Tog bort " + result.get("name"));
          this.detailedView = false;
          this.unselectAll();
          if (result.className == "Container") {
            this.changeContainer(this.currentObject);
          } else {
            if (this.currentObject && this.currentObject.id == result.id) {
              this.goToParent();
            } else {
              this.things.splice(
                this.things.findIndex(x => x.obj.id == result.id),
                1
              );
              this.loadingDialog = false;
            }
          }
        })
        .catch(err => {
          this.error("Error när en sak skulle tas bort!");
          console.error("Error while deleting Thing", err);
          this.loadingDialog = false;
        })
        .finally(() => {
          this.selectedObjectAction = () => {};
        });
    };
    this.warningMsg = "Vill du verkligen ta bort '" + obj.get("name") + "'?";
  }

  deleteSelectedObjects() {
    this.warningDialog = true;
    this.warningMsg =
      "Är du säker på att du vill tabort " +
      this.selectedObjects.length +
      " object?";
    this.selectedObjectAction = () => {
      this.warningDialog = false;
      this.loadingDialog = true;
      let promises = this.selectedObjects.map(x =>
        this.$store.dispatch("deleteObject", x)
      );
      Promise.all(promises)
        .then(results => {
          this.info("Tog bort " + results.length + " objekt.");
        })
        .catch(err => {
          this.error("Kunde inte tabort objecten. \n" + err);
          console.error("Error while deleting Thing", err);
        })
        .finally(() => {
          this.selectedObjectAction = () => {};
          this.changeContainer(this.currentObject);
          this.selectedObjectAction = () => {};
        });
    };
  }

  public updOrCrEvent(obj: any, final = true) {
    if (obj) {
      console.log(obj);
      if (obj.obj.className == "Thing") {
        if (this.currentObject && this.currentObject.id === obj.obj.id) {
          this.currentObject = obj.obj;
          this.updateThingInfo();
        } else {
          const foundIndex = this.things.findIndex(x => x.obj.id == obj.obj.id);
          if (foundIndex >= 0) {
            this.things.splice(foundIndex, 1, obj);
            if (final) {
              this.objSelected(obj);
              this.createDialog = false;
              this.detailedView = true;
              this.info("Uppdaterade " + obj.obj.get("name"));
            }
          } else {
            this.things.push(obj);
            console.log(obj.tags);
            if (final) {
              this.info("Skapade en ny sak!");
            }
          }
        }
      } else if (obj.obj.className == "Container") {
        const foundIndex = this.containers.findIndex(
          x => x.obj.id == obj.obj.id
        );
        if (foundIndex >= 0) {
          this.containers.splice(foundIndex, 1, obj);
          if (final) {
            this.objSelected(obj);
            this.createDialog = false;
            this.detailedView = true;
            this.info("Uppdaterade " + obj.obj.get("name"));
          }
        } else {
          this.containers.push(obj);
          if (final) {
            this.info("Skapade ny container!");
          }
        }
      }
    } else {
      this.error(
        "Fel uppstod! Kunde inte skapa eller uppdatera objekt. Information har kanske inte sparats."
      );
    }
  }
}
</script>

<style scoped lang="scss">
.browser {
  height: calc(100% - 48px);
}

.breadcrumb {
  display: flex;
  flex-direction: row;
  padding: 10px 23px;

  p {
    margin: 0 5px;
  }
}

.no-data {
  padding: 30px 0;
  text-align: center;
}

.spacer {
  margin: 10px 0;
  padding: 20px 40px 10px 40px;
}

.wrapper-view {
  display: flex;
  flex-direction: row;
  overflow: hidden;
  height: 100%;
}

.main-view {
  width: 100%;
  overflow-y: auto;
}

.side-view {
  width: 35%;
  min-width: 400px;
  overflow: hidden auto;

  @media screen and (max-width: 700px) {
    width: 100vw;
    min-width: 100vw;
    margin: 0;
  }
}
</style>