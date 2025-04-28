<template>
  <div>
    <vue-element-loading :active="isSubmit" />
    <v-card>
      <v-card-title>
        <v-icon color="primary">
          {{ getIcon(userType) }}
        </v-icon>
        <span class="pl-2">{{ (userType == 'dsp') ? 'Delivery Service Providers' : capitalizeFirstLetter(userType) }}</span>
      </v-card-title>
      <v-spacer></v-spacer>
      <create-button @create="showUserDialog" v-if="userType === 'students' || userType === 'drivers' || userType === 'guardians' || userType === 'schools' || userType === 'dsp'"></create-button>
      <v-tabs v-model="active_tab" show-arrows class="my-2">
        <v-tab v-for="tab in tabs" :key="tab.idx">
          <v-icon size="20" class="me-3">
            {{ tab.icon }}
          </v-icon>
          <span>{{ tab.title }}</span>
        </v-tab>
      </v-tabs>
      <!-- tabs item -->
      <v-tabs-items v-model="active_tab">
        <!-- active -->
        <v-tab-item>
          <users-table
              :users="activeUsers"
              :userType="userType"
              :tab="active_tab"
              @view-user="viewUser"
              @edit-user="editUser"
              @suspend-user="suspendActivateUser"
              @unassign-bus="unAssignBus"
              @assign-bus="assignBus"
              @login-as-school="loginAsSchool"
          ></users-table>
        </v-tab-item>

        <!-- suspended -->
        <v-tab-item>
          <users-table
              :users="suspendedUsers"
              :userType="userType"
              :tab="active_tab"
              @view-user="viewUser"
              @edit-user="editUser"
              @suspend-user="suspendActivateUser"
              @unassign-bus="unAssignBus"
              @assign-bus="assignBus"
              @login-as-school="loginAsSchool"
          ></users-table>
        </v-tab-item>

        <v-tab-item>
          <users-table
              v-if="userType === 'drivers' || userType === 'students'"
              :users="underReviewUsers"
              :userType="userType"
              :tab="active_tab"
              @view-user="viewUser"
          ></users-table>
        </v-tab-item>
      </v-tabs-items>
    </v-card>
    <v-dialog v-if="selectedDriver" v-model="busesDialog" max-width="390">
      <v-card>
        <v-card-title class="text-h5">
          Select bus for '{{ selectedDriver.name }}'
        </v-card-title>

        <v-card-text>
          <v-list dense>
            <v-subheader>Buses</v-subheader>
            <v-list-item-group>
              <v-list-item
                  v-for="(bus, i) in availableBuses"
                  :key="i"
              >
                <v-list-item-content
                    @click="assignBusToDriver(bus)"
                >
                  <v-list-item-title v-text="'License: ' + bus.license"
                  ></v-list-item-title>
                  <v-list-item-subtitle
                      v-text="'Capacity: ' + bus.capacity"
                  ></v-list-item-subtitle>
                </v-list-item-content>
              </v-list-item>
            </v-list-item-group>
          </v-list>
        </v-card-text>
        <v-container style="height: 400px">
          <v-row
              v-show="loadingBuses"
              class="fill-height"
              align-content="center"
              justify="center"
          >
            <v-col class="text-subtitle-1 text-center" cols="12">
              Please wait ...
            </v-col>
            <v-col cols="6">
              <v-progress-linear
                  :active="loadingBuses"
                  color="primary"
                  indeterminate
                  rounded
                  height="6"
              ></v-progress-linear>
            </v-col>
          </v-row>
        </v-container>
        <v-card-actions>
          <v-spacer></v-spacer>

          <v-btn color="green darken-1" text @click="closeBusDialog">
            Close
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-row justify="center">
      <v-dialog v-model="studentDialog" persistent max-width="700px">
        <v-form ref="studentForm" @submit.prevent="createStudent" v-model="valid" lazy-validation>
          <v-card>
            <v-card-title>
              <span class="text-h5">Add Student</span>
            </v-card-title>
            <v-card-text>
              <v-container>
                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="studentForm.name"
                        label="Student Name*"
                        hint="name of the student"
                        required
                        :rules="[v => !!v || 'Name is required']"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="studentForm.student_identification"
                        label="Student Identification*"
                        hint="Student Identification"
                        required
                        :rules="[v => !!v || 'Student ID is required']"
                    ></v-text-field>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="studentForm.age"
                        label="Age"
                        hint="Age"
                        type="number"
                        min="1"
                        required
                        :rules="[v => !!v || 'Age is required', v => v > 0 || 'Age must be positive']"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-file-input
                        v-model="studentForm.pic"
                        accept="image/*"
                        label="Student Photo"
                        @change="handleFileUpload"
                    ></v-file-input>
                  </v-col>
                </v-row>
                <v-row >
                  <!-- School Select Dropdown -->
                  <v-col cols="12" sm="12" md="12">
                    <v-select
                        v-model="studentForm.school_id"
                        :items="showSchools"
                        label="Select School"
                        item-value="id"
                        item-text="name"
                        :rules="[v => !!v || 'School is required']"
                    ></v-select>
                  </v-col>

                  <!-- Guardian Select Dropdown (School-wise) -->
                  <v-col cols="12" sm="12" md="12">
                    <v-select
                        v-model="studentForm.parent_id"
                        :items="showGuardiansBySchool"
                        label="Select Guardian"
                        item-value="id"
                        item-text="name"
                        :rules="[v => !!v || 'Guardian is required']"
                    ></v-select>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="12" md="12">
                    <v-textarea
                        v-model="studentForm.notes"
                        label="Notes*"
                        hint="Notes: e.g. class,year, etc"
                        required
                        :rules="[v => !!v || 'Notes are required']"
                    ></v-textarea>
                  </v-col>
                </v-row>
              </v-container>
            </v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn
                  color="blue darken-1"
                  text
                  @click="studentDialog = false"
              >
                Close
              </v-btn>
              <v-btn
                  color="blue darken-1"
                  text
                  type="submit"
              >
                Save
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-form>
      </v-dialog>
      <v-dialog v-model="driverDialog" persistent max-width="800px">
        <v-form ref="driverForm" @submit.prevent="createDriver" v-model="valid" lazy-validation>
          <v-card>
            <v-card-title>
              <span class="text-h5">Create Driver</span>
            </v-card-title>
            <v-card-text>
              <v-container>
                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="email"
                        label="Email Address*"
                        hint="Email Address"
                        required
                        :rules="emailRules"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="phone_number"
                        label="Phone Number*"
                        hint="Phone Number"
                        required
                        :rules="phoneRules"
                    ></v-text-field>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        type="password"
                        v-model="password"
                        label="Password"
                        hint="Password"
                        required
                        :rules="passwordRules"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        type="password"
                        v-model="confirm_password"
                        label="Confirm Password"
                        hint="Confirm Password"
                        required
                        :rules="[v => !!v || 'Confirm Password is required',
                                                    v => v === password || 'Passwords must match']"
                    ></v-text-field>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="first_name"
                        label="First Name*"
                        hint="First Name"
                        required
                        :rules="[v => !!v || 'First Name is required']"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="last_name"
                        label="Last Name*"
                        hint="Last Name"
                        required
                        :rules="[v => !!v || 'Last Name is required']"
                    ></v-text-field>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="12" md="12">
                    <v-textarea
                        v-model="address"
                        label="Address*"
                        hint="Address"
                        :rules="[v => !!v || 'Address is required']"
                    ></v-textarea>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="12" md="12">
                    <v-text-field
                        v-model="document_number"
                        label="Driving License Number*"
                        hint="Driving License Number"
                        :rules="[v => !!v || 'License Number is required']"
                    ></v-text-field>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="12" md="12">
                    <v-text-field
                        v-model="document_name"
                        label="Document Name*"
                        hint="Document Name"
                        :rules="[v => !!v || 'Document Name is required']"
                    ></v-text-field>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="12" md="12">
                    <v-select
                        label="Select City"
                        :items="cityLists"
                        item-text="name"
                        item-value="id"
                        variant="outlined"
                        v-model="selectedCity"
                        @change="getDistrictByCity"
                        :rules="[v => !!v || 'City is required']"
                    ></v-select>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="12" md="12">
                    <v-select
                        label="Select District"
                        :items="districtList"
                        item-text="name"
                        item-value="id"
                        v-model="selectedDistrict"
                        variant="outlined"
                        :rules="[v => !!v || 'District is required']"
                    ></v-select>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="12" md="12">
                    <v-text-field
                        type="date"
                        v-model="expiry_date"
                        label="Expiry Date*"
                        hint="Expiry Date"
                        :rules="[v => !!v || 'Expiry Date is required']"
                    ></v-text-field>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="12" md="12">
                    <v-file-input
                        label="Document Image"
                        v-model="document_image"
                        @change="handleFileUpload"
                        :rules="[v => !!v || 'Document Image is required']"
                    ></v-file-input>
                    <v-progress-linear
                        v-if="uploadPercentage !== null"
                        color="yellow-darken-2"
                        indeterminate
                    ></v-progress-linear>
                  </v-col>
                </v-row>
              </v-container>
            </v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn
                  color="blue darken-1"
                  text
                  @click="driverDialog = false"
              >
                Close
              </v-btn>
              <v-btn
                  color="blue darken-1"
                  text
                  type="submit"
              >
                Save
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-form>
      </v-dialog>

      <v-dialog v-model="parentDialog" persistent max-width="700px">
        <v-form ref="parentForm" @submit.prevent="createParent" v-model="valid" lazy-validation>
          <v-card>
            <v-card-title>
              <span class="text-h5">Create Guardian</span>
            </v-card-title>
            <v-card-text>
              <v-container>
                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="email"
                        label="Email Address*"
                        hint="Email Address"
                        required
                        :rules="emailRules"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="phone_number"
                        label="Phone Number*"
                        hint="Phone Number"
                        required
                        :rules="phoneRules"
                    ></v-text-field>
                  </v-col>
                </v-row>

                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        type="password"
                        v-model="password"
                        label="Password"
                        hint="Password"
                        required
                        :rules="passwordRules"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        type="password"
                        v-model="confirm_password"
                        label="Confirm Password"
                        hint="Confirm Password"
                        required
                        :rules="[v => !!v || 'Confirm Password is required',
                                                    v => v === password || 'Passwords must match']"
                    ></v-text-field>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="first_name"
                        label="First Name*"
                        hint="First Name"
                        required
                        :rules="[v => !!v || 'First Name is required']"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="last_name"
                        label="Last Name*"
                        hint="Last Name"
                        required
                        :rules="[v => !!v || 'Last Name is required']"
                    ></v-text-field>
                  </v-col>
                </v-row>
              </v-container>
            </v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn
                  color="blue darken-1"
                  text
                  @click="parentDialog = false"
              >
                Close
              </v-btn>
              <v-btn
                  color="blue darken-1"
                  text
                  type="submit"
              >
                Save
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-form>
      </v-dialog>
      <v-dialog v-model="schoolDialog" persistent max-width="700px">
        <v-form ref="schoolForm" @submit.prevent="createSchool" v-model="valid" lazy-validation>
          <v-card>
            <v-card-title>
              <span class="text-h5">Add School</span>
            </v-card-title>
            <v-card-text>
              <v-container>
                <div class="v-text-field__slot">
                  <GmapAutocomplete
                      id="stop-address"
                      @place_changed="setPlace"
                      placeholder="School Address"
                      class="form-control"
                  />
                </div>

                <v-row>
                  <v-col cols="12" sm="12" md="12">
                    <v-text-field
                        v-model="schoolForm.name"
                        label="Name*"
                        hint="Name"
                        required
                        :rules="[v => !!v || 'Name is required']"
                    ></v-text-field>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="schoolForm.phone"
                        label="Phone Number*"
                        hint="Phone Number"
                        required
                        :rules="phoneRules"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="schoolForm.email"
                        label="Email*"
                        hint="Email"
                        required
                        :rules="emailRules"
                    ></v-text-field>
                  </v-col>
                </v-row>

                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        type="password"
                        v-model="schoolForm.password"
                        label="Password"
                        hint="Password"
                        required
                        :rules="passwordRules"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        type="password"
                        v-model="schoolForm.confirm_password"
                        label="Confirm Password"
                        hint="Confirm Password"
                        required
                        :rules="[v => !!v || 'Confirm Password is required',
                                                    v => v === schoolForm.password || 'Passwords must match']"
                    ></v-text-field>
                  </v-col>
                </v-row>
              </v-container>
            </v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn
                  color="blue darken-1"
                  text
                  @click="schoolDialog = false"
              >
                Close
              </v-btn>
              <v-btn
                  color="blue darken-1"
                  text
                  type="submit"
              >
                Save
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-form>
      </v-dialog>
      <v-dialog v-model="dspDialog" persistent max-width="700px">
        <v-form ref="dspForm" @submit.prevent="createDsp" v-model="valid" lazy-validation>
          <v-card>
            <v-card-title>
              <span class="text-h5">Add DSP Company</span>
            </v-card-title>
            <v-card-text>
              <v-container>
                <v-row>
                  <v-col cols="12" sm="12" md="12">
                    <v-text-field
                        v-model="dspForm.name.value"
                        label="Name*"
                        hint="Name"
                        :rules="dspForm.name.rules"
                    ></v-text-field>
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="dspForm.email.value"
                        label="Email Address*"
                        hint="Email Address"
                        :rules="dspForm.email.rules"
                        required
                    ></v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        v-model="dspForm.phone.value"
                        label="Phone Number*"
                        hint="Phone Number"
                        :rules="dspForm.phone.rules"
                    ></v-text-field>
                  </v-col>
                </v-row>

                <v-row>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        type="password"
                        v-model="dspForm.password.value"
                        label="Password"
                        hint="Password"
                        :rules="dspForm.password.rules">
                    </v-text-field>
                  </v-col>
                  <v-col cols="12" sm="6" md="6">
                    <v-text-field
                        type="password"
                        v-model="dspForm.confirm_password.value"
                        label="Confirm Password"
                        hint="Confirm Password"
                        :rules="dspForm.confirm_password.rules">
                    </v-text-field>
                  </v-col>
                </v-row>
              </v-container>
            </v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn
                  color="blue darken-1"
                  text
                  @click="dspDialog = false"
              >
                Close
              </v-btn>
              <v-btn
                  color="blue darken-1"
                  text
                  type="submit"
              >
                Save
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-form>
      </v-dialog>
    </v-row>
  </div>
</template>

<script>
import usersTable from "./users-table.vue";
import { GmapAutocomplete } from 'vue2-google-maps';

import {
  mdiAccountCheck,
  mdiAccountOff,
  mdiAirplane,
  mdiMotionPause,
  mdiAccountClock,
  mdiAccountQuestion,
} from "@mdi/js";

import { ref } from "@vue/composition-api";
import { Keys } from "/src/config.js";
import VueElementLoading from "vue-element-loading";
import CreateButton from "@/components/CreateButton";
import auth from "@/services/AuthService";

export default {
  name: 'GooglePlacesAutocomplete',
  components: {
    VueElementLoading,
    usersTable,
    CreateButton
  },
  data() {
    return {
      apiKey: Keys.GOOGLE_MAPS_API_KEY,
      currentPlace: null,
      autocomplete: null,
      search: '',
      selectedAddress: '',
      studentName:"",
      studentId:"",
      school:"",
      notes:"",
      userDialog: false,
      driverDialog:false,
      parentDialog:false,
      studentDialog:false,
      schoolDialog:false,
      dspDialog:false,
      userType: "",
      valid: true,
      selectedDistrict: '',
      selectedCity: '',
      users: [],
      activeUsers: [],
      suspendedUsers: [],
      underReviewUsers: [],
      availableBuses: [],
      dialog: false,
      busesDialog: false,
      loadingBuses: false,
      isLoading: false,
      isSubmit: false,
      selectedUser: null,
      selectedDriver: null,
      tabs: [],
      cityLists:[],
      districtList:[],
      extendedTabs: [
        { idx: 0, title: "Active", icon: mdiAirplane },
        { idx: 1, title: "Suspended", icon: mdiMotionPause },
        { idx: 2, title: "Under Review", icon: mdiAccountClock },
      ],
      defaultTabs: [
        { idx: 0, title: "Active", icon: mdiAirplane },
        { idx: 1, title: "Suspended", icon: mdiMotionPause },
      ],
      active_tab: null,

      first_name:'',
      last_name:'',
      phone_number:'',
      address:'',
      password:'',
      email:'',
      confirm_password:'',
      document_number:'',
      document_name:'',
      expiry_date:'',
      document_image:null,
      uploadPercentage: null,
      guardianList : [],
      schoolList: [],
      schoolByGuardianList: [],

      // Validation rules
      emailRules: [
        v => !!v || 'Email is required',
        v => /.+@.+\..+/.test(v) || 'Email must be valid',
      ],
      phoneRules: [
        v => !!v || 'Phone is required',
        v => /^[0-9]{10,15}$/.test(v) || 'Phone must be valid (10-15 digits)',
      ],
      passwordRules: [
        v => !!v || 'Password is required',
        v => (v && v.length >= 6) || 'Password must be at least 6 characters',
      ],

      studentForm:{
        name:'',
        age:'',
        student_identification:'',
        school_id:null,
        parent_id:null,
        device_id:'',
        pic:'',
        role_id:null
      },
      schoolForm:{
        token : '',
        role:2,
        email:'',
        phone:'',
        latitude:null,
        longitude:null,
        device_name:'',
        address: '',
        password:'',
        confirm_password:'',
        name: ''
      },
      dspForm:{
        token : '',
        role:7,
        name : {
          value:'',
          rules:[
            v => !!v || 'Name is required',
          ]
        },
        email:{
          value:'',
          rules:[
            v => !!v || 'Email is required',
            v => /.+@.+\..+/.test(v) || 'Email must be valid',
          ]
        },
        phone:{
          value:'',
          rules:[
            v => !!v || 'Phone number is required',
            v => /^[0-9]{10,15}$/.test(v) || 'Phone number must be valid',
          ]
        },
        device_name:'',
        password:{
          value: '',
          rules: [
            v => !!v || 'Password is required',
            v => v.length >= 6 || 'Password must be at least 6 characters',
          ]
        },
        confirm_password:{
          value: '',
          rules: [
            v => !!v || 'Confirm password is required',
            v => v === this.dspForm.password.value || 'Passwords must match',
          ]
        }
      },
    };
  },
  computed: {
    showGuardians() {
      return this.guardianList.map(item => ({
        id: item.id,
        name: item.name
      }));
    },
    showSchools() {
      return this.schoolList.map(school => ({
        id: school.id,
        name: school.name
      }));
    },
    showGuardiansBySchool() {
      return this.schoolByGuardianList.map(guardian => ({
        id: guardian.id,
        name: guardian.name
      }));
    }
  },
  watch: {
    $route(to, from) {
      this.userType = to.name;
      console.log("to.name",this.userType);
      this.updateTabs();
      if (this.userType === "drivers") {
        this.active_tab = parseInt(localStorage.tabIdxDrivers) || 0;
      } else if (this.userType === "students") {
        this.active_tab = parseInt(localStorage.tabIdxStudents) || 0;
        this.getGuardianList();
        if (this.studentForm?.school_id) {
          this.getGuardiansBySchool();
        }
      } else if (this.userType === "guardians") {
        this.active_tab = parseInt(localStorage.tabIdxParents) || 0;
      } else if (this.userType === "schools") {
        this.active_tab = parseInt(localStorage.tabIdxSchools) || 0;
      }
      this.loadUsers();
    },
    active_tab: function (newVal, oldVal) {
      if (this.userType === "drivers") {
        localStorage.tabIdxDrivers = newVal;
      } else if (this.userType === "guardians") {
        localStorage.tabIdxParents = newVal;
      } else if (this.userType === "students") {
        localStorage.tabIdxStudents = newVal;
      } else if (this.userType === "schools") {
        localStorage.tabIdxSchools = newVal;
      }
    },
    "studentForm.school_id": function() {
      this.getGuardiansBySchool();
      this.studentForm.parent_id = null;
    }
  },
  mounted() {
    this.loadGoogleMapsAPI()
        .then(() => {
          this.initAutocomplete();
        })
        .catch(error => {
          console.error('Google Maps API load error:', error);
        });

    this.userType = this.$router.currentRoute.name;
    this.updateTabs();
    if (this.userType === "drivers") {
      this.active_tab = parseInt(localStorage.tabIdxDrivers) || 0;
    } else if (this.userType === "guardians") {
      this.active_tab = parseInt(localStorage.tabIdxParents) || 0;
    } else if (this.userType === "students") {
      this.active_tab = parseInt(localStorage.tabIdxStudents) || 0;
      this.getSchools();
      this.getGuardianList();
    } else if (this.userType === "schools") {
      this.active_tab = parseInt(localStorage.tabIdxSchools) || 0;
    }
    this.loadUsers();
    this.loadCity();

    if (this.userType === "drivers" && localStorage.userRole == 2) {
      this.loadAvailableBuses();
    }
  },
  methods: {
    loadGoogleMapsAPI() {
      return new Promise((resolve, reject) => {
        if (window.google && window.google.maps && window.google.maps.places) {
          resolve();
        } else {
          // const script = document.createElement('script');
          // script.src = `https://maps.googleapis.com/maps/api/js?key=AIzaSyBzUewKNxLW4q92Cv6Uarc238TX7I1AKKM&libraries=places`;
          // script.async = true;
          // script.defer = true;
          // script.onload = resolve;
          // script.onerror = reject;
          // document.head.appendChild(script);
        }
      });
    },
    initAutocomplete() {
      // This will be called after the Google Maps API is loaded
      // You can initialize your autocomplete here if needed
    },
    setPlace(place) {
      this.currentPlace = place;
      this.updateStopFromPlace(place);
      this.addStopMarker();
    },
    updateStopFromPlace(place) {
      this.stop.place_id = place.place_id;
      this.stop.address = place.formatted_address;
      this.stop.lat = place.geometry.location.lat();
      this.stop.lng = place.geometry.location.lng();
    },
    addStopMarker() {
      if (this.stop) {
        const position = {
          lat: parseFloat(this.stop.lat),
          lng: parseFloat(this.stop.lng),
        };
        let marker = {
          place_id: this.stop.place_id,
          position: position,
          infoText:
              "<strong>" + this.stop.name + "</strong><br/>" + this.stop.address,
        };
        this.markers = [];
        this.markers.push(marker);
        this.center = position;
      }
    },
    updateTabs() {
      if (this.userType === "drivers" || this.userType === "students") {
        this.tabs = this.extendedTabs;
      } else {
        this.tabs = this.defaultTabs;
      }
    },
    capitalizeFirstLetter(string) {
      return string.charAt(0).toUpperCase() + string.slice(1);
    },
    getIcon(userType) {
      switch (userType) {
        case "schools":
          return "mdi-school";
        case "students":
          return "mdi-badge-account-outline";
        case "drivers":
          return "mdi-account-tie-hat";
        case "guardians":
          return "mdi-account-group-outline";
        default:
          return "mdi-account";
      }
    },
    loadUsers() {
      this.isLoading = true;
      let url = "/users/all-" + this.userType;
      this.users = [];
      axios
          .get(url)
          .then((response) => {
            this.users = response.data;
            console.log("Users",this.users)
            this.activeUsers = this.users.filter(
                (user) => user.status_id === 1
            );
            this.suspendedUsers = this.users.filter(
                (user) => user.status_id === 3
            );
            this.underReviewUsers = this.users.filter(
                (user) => user.status_id === 4
            );
          })
          .catch((error) => {
            this.$notify({
              title: "Error",
              text: "Error while retrieving users",
              type: "error",
            });
            console.log(error);
            auth.checkError(error.response.data.message, this.$router, this.$swal);
          })
          .finally(() => {
            this.isLoading = false;
          });
    },
    loginAsSchool(school) {
      this.$swal
          .fire({
            title: "Login as school",
            text: `Are you sure to login as '${school.name}'?`,
            icon: "warning",
            showCancelButton: true,
            confirmButtonText: "Yes, login",
          })
          .then((result) => {
            if (result.isConfirmed) {
              this.loginAsSchoolServer(school);
            }
          });
    },
    loginAsSchoolServer(school) {
      this.isLoading = true;
      axios
          .post("/auth/login-from-admin-to-school", {
            school_id: school.id,
            device_name: `${this.$browserDetect.meta.name}- v${this.$browserDetect.meta.version}`
          })
          .then((response) => {
            this.isLoading = false;
            this.$notify({
              title: "Success",
              text: "Logged in as " + school.name,
              type: "success",
            });
            const ourToken = response.data.token;
            let user = response.data.user_data;
            const userRole = user.role_id;
            const freshToken = ourToken.split('|')[1];
            localStorage.setItem('freshToken', freshToken);
            localStorage.setItem('userRole', userRole);
            axios.defaults.headers.common['Authorization'] = `Bearer ${freshToken}`;
            this.$router.push({
              name: "school-dashboard",
            });
          })
          .catch((error) => {
            this.isLoading = false;
            this.$notify({
              title: "Error",
              text: "Error",
              type: "error",
            });
            console.log(error);
            this.$swal("Error", error.response.data.message, "error");
          });
    },
    viewUser(user) {
      let userType = this.userType.slice(0, -1);
      let routeName = "view-" + userType;
      this.$router.push({
        name: routeName,
        params: {
          user_id: user.id,
        },
      });
    },
    editUser(user) {
      let userType = this.userType.slice(0, -1);
      let routeName = "edit-" + userType;
      this.$router.push({
        name: routeName,
        params: {
          user_id: user.id,
        },
      });
    },
    suspendActivateUser(user, index) {
      let userType = this.userType.slice(0, -1);
      const action = user.status_id != 1 ? "activate" : "suspend";

      this.$swal
          .fire({
            title: `${action.charAt(0).toUpperCase() + action.slice(1)} ${userType}`,
            text: `Are you sure to ${action} '${user.name}'?`,
            icon: user.status_id != 1 ? "success" : "error",
            showCancelButton: true,
            confirmButtonText: "Yes",
          })
          .then((result) => {
            if (result.isConfirmed) {
              this.suspendActivateUserServer(user, index);
            }
          });
    },
    suspendActivateUserServer(user, indexx) {
      this.isSubmit = true;
      axios
          .post("/users/suspend-activate", {
            user_id: user.id,
          })
          .then((response) => {
            this.isSubmit = false;
            let index = this.users.indexOf(user);
            this.users[index].status_id = user.status_id != 1 ? 1 : 3;
            this.activeUsers = this.users.filter(
                (user) => user.status_id === 1
            );
            this.suspendedUsers = this.users.filter(
                (user) => user.status_id === 3
            );
            this.$notify({
              title: "Success",
              text: `User ${user.status_id != 1 ? "activated" : "suspended"}`,
              type: "success",
            });
          })
          .catch((error) => {
            this.isSubmit = false;
            this.$notify({
              title: "Error",
              text: "Error",
              type: "error",
            });
            this.$swal("Error", error.response.data.message, "error");
          });
    },
    loadAvailableBuses() {
      this.loadingBuses = true;
      this.availableBuses = [];
      axios
          .get("/drivers/available-buses")
          .then((response) => {
            this.availableBuses = response.data;
          })
          .catch((error) => {
            this.$notify({
              title: "Error",
              text: "Error while retrieving buses",
              type: "error",
            });
            this.$swal("Error", error.response.data.message, "error");
          })
          .finally(() => {
            this.loadingBuses = false;
          });
    },
    assignBus(item) {
      this.selectedDriver = item;
      this.busesDialog = true;
      this.loadAvailableBuses();
    },
    closeBusDialog() {
      this.busesDialog = false;
      this.loadingBuses = false;
      this.availableBuses = [];
    },
    assignBusToDriver(bus) {
      this.loadingBuses = true;
      axios
          .post("/drivers/assign-bus", {
            driver_id: this.selectedDriver.id,
            bus_id: bus.id,
          })
          .then((response) => {
            this.loadingBuses = false;
            this.selectedDriver.bus = bus;
            this.closeBusDialog();
            this.$notify({
              title: "Success",
              text: "Bus assigned to driver",
              type: "success",
            });
          })
          .catch((error) => {
            this.isSubmit = false;
            this.$notify({
              title: "Error",
              text: "Error",
              type: "error",
            });
            this.$swal("Error", error.response.data.message, "error");
          })
          .finally(() => {
            this.closeBusDialog();
          });
    },
    unAssignBus(item) {
      this.$swal
          .fire({
            title: "Un-assign bus",
            text: `Are you sure to un-assign the driver '${item.name}' from the bus '${item.bus.license}'? You won't be able to revert this!`,
            icon: "error",
            showCancelButton: true,
            confirmButtonText: "Yes, delete it!",
          })
          .then((result) => {
            if (result.isConfirmed) {
              this.unassignBusFromDriver(item);
            }
          });
    },
    unassignBusFromDriver(driver) {
      this.isLoading = true;
      axios
          .post("/drivers/unassign-bus", {
            driver_id: driver.id,
          })
          .then((response) => {
            this.isLoading = false;
            driver.bus = null;
            this.$notify({
              title: "Success",
              text: "Bus unassigned from driver",
              type: "success",
            });
          })
          .catch((error) => {
            this.isSubmit = false;
            this.$notify({
              title: "Error",
              text: "Error",
              type: "error",
            });
            this.$swal("Error", error.response.data.message, "error");
          })
          .finally(() => {
            this.closeBusDialog();
          });
    },
    showUserDialog() {
      if(this.userType === 'students'){
        this.studentDialog = true;
      }
      else if(this.userType === 'drivers'){
        this.driverDialog = true;
      }
      else if(this.userType === 'guardians'){
        this.parentDialog = true;
      }
      else if(this.userType === 'schools'){
        this.schoolDialog = true;
      }
      else if(this.userType === 'dsp'){
        this.dspDialog = true;
      }
    },
    handleFileUpload(file) {
      if (!file) return;

      this.uploadPercentage = 0;
      const reader = new FileReader();

      reader.onload = () => {
        if(this.userType === 'drivers'){
          this.document_image = reader.result;
        } else {
          this.studentForm.pic = reader.result;
        }
        this.uploadPercentage = null;
      };

      reader.onprogress = (event) => {
        if (event.lengthComputable) {
          this.uploadPercentage = Math.round((event.loaded / event.total) * 100);
        }
      };

      reader.onerror = () => {
        this.uploadPercentage = null;
        this.$notify({
          title: "Error",
          text: "Error reading file",
          type: "error",
        });
      };

      reader.readAsDataURL(file);
    },
    async createDriver() {
      if (this.$refs.driverForm.validate()) {
        this.isLoading = true;
        try {
          const payload = {
            email: this.email,
            password: this.password,
            notify: this.$notify,
          };

          const response = await auth.registerDriver(payload);
          if (!response) {
            throw new Error("Registration failed");
          }

          const driverData = {
            submit: true,
            first_name: this.first_name,
            last_name: this.last_name,
            phone_number: this.phone_number,
            address: this.address,
            email: this.email,
            password: this.password,
            license_number: this.document_number,
            device_name: response.device_name,
            token: response.token,
            city_id: this.selectedCity,
            district_id: this.selectedDistrict,
            documents: [
              {
                document_image: this.document_image,
                document_name: this.document_name,
                expiry_date: this.expiry_date,
                document_number: this.document_number
              },
            ],
          };

          await axios.post("/auth/create-driver", driverData);

          this.$notify({
            title: "Success",
            text: "Driver added successfully!",
            type: "success",
          });
          this.driverDialog = false;
          this.loadUsers();
        } catch (error) {
          console.error(error);
          this.$notify({
            title: "Error",
            text: error.response?.data?.message || "Failed to create driver",
            type: "error",
          });
        } finally {
          this.isLoading = false;
        }
      }
    },
    async createParent() {
      if (this.$refs.parentForm.validate()) {
        this.isLoading = true;
        try {
          const response = await axios.post("/users/add-guardian", {
            name: this.first_name + ' ' + this.last_name,
            phone_number: this.phone_number,
            email: this.email,
            password: this.password,
          });

          this.$notify({
            title: "Success",
            text: "Guardian added successfully!",
            type: "success",
          });
          this.parentDialog = false;
          this.loadUsers();
        } catch (error) {
          console.error(error);
          this.$notify({
            title: "Error",
            text: error.response?.data?.message || "Failed to create guardian",
            type: "error",
          });
        } finally {
          this.isLoading = false;
        }
      }
    },
    getGuardianList() {
      this.isLoading = true;
      axios.get("/users/guardians-list")
          .then((response) => {
            this.guardianList = response.data;
          })
          .catch((error) => {
            this.$notify({
              title: "Error",
              text: "Error while retrieving guardians",
              type: "error",
            });
            auth.checkError(error.response.data.message, this.$router, this.$swal);
          })
          .finally(() => {
            this.isLoading = false;
          });
    },
    async getSchools() {
      this.isLoading = true;
      try {
        const response = await axios.get("/users/all-schools");
        this.schoolList = response.data;
      } catch (error) {
        this.$notify({
          title: "Error",
          text: "Error while retrieving schools",
          type: "error",
        });
        auth.checkError(error.response.data.message, this.$router, this.$swal);
      } finally {
        this.isLoading = false;
      }
    },
    async getGuardiansBySchool() {
      if (!this.studentForm.school_id) {
        this.schoolByGuardianList = [];
        return;
      }

      this.isLoading = true;
      try {
        const response = await axios.get(`/users/guardians-list/${this.studentForm.school_id}`);
        this.schoolByGuardianList = response.data;
      } catch (error) {
        this.$notify({
          title: "Error",
          text: "Error while retrieving guardians by school",
          type: "error",
        });
        auth.checkError(error.response.data.message, this.$router, this.$swal);
      } finally {
        this.isLoading = false;
      }
    },
    async createStudent() {
      if (this.$refs.studentForm.validate()) {
        this.isLoading = true;
        try {
          await axios.post("/users/add-edit-student-from-school", this.studentForm);
          this.$notify({
            title: "Success",
            text: "Student added successfully!",
            type: "success",
          });
          this.studentDialog = false;
          this.loadUsers();
        } catch (error) {
          console.error(error);
          this.$notify({
            title: "Error",
            text: error.response?.data?.message || "Failed to create student",
            type: "error",
          });
        } finally {
          this.isLoading = false;
        }
      }
    },
    async createSchool() {
      if (this.$refs.schoolForm.validate()) {
        this.isLoading = true;
        try {
          const payload = {
            email: this.schoolForm.email,
            password: this.schoolForm.password,
            notify: this.$notify,
          };

          const response = await auth.registerDriver(payload);
          if (!response) {
            throw new Error("Registration failed");
          }

          this.schoolForm.token = response.token;
          this.schoolForm.device_name = response.device_name;

          await axios.post("/auth/loginViaToken", this.schoolForm);

          this.$notify({
            title: "Success",
            text: "School added successfully!",
            type: "success",
          });
          this.schoolDialog = false;
          this.loadUsers();
        } catch (error) {
          console.error(error);
          this.$notify({
            title: "Error",
            text: error.response?.data?.message || "Failed to create school",
            type: "error",
          });
        } finally {
          this.isLoading = false;
        }
      }
    },
    async createDsp() {
      if (this.$refs.dspForm.validate()) {
        this.isLoading = true;
        try {
          const payload = {
            email: this.dspForm.email.value,
            password: this.dspForm.password.value,
            notify: this.$notify,
          };

          const response = await auth.registerDriver(payload);
          if (!response) {
            throw new Error("Registration failed");
          }

          const dspData = {
            token: response.token,
            role: this.dspForm.role,
            email: this.dspForm.email.value,
            phone: this.dspForm.phone.value,
            device_name: response.device_name,
            password: this.dspForm.password.value,
            confirm_password: this.dspForm.confirm_password.value,
            name: this.dspForm.name.value
          };

          await axios.post("/auth/loginViaToken", dspData);

          this.$notify({
            title: "Success",
            text: "DSP added successfully!",
            type: "success",
          });
          this.dspDialog = false;
          this.loadUsers();
        } catch (error) {
          console.error(error);
          this.$notify({
            title: "Error",
            text: error.response?.data?.message || "Failed to create DSP",
            type: "error",
          });
        } finally {
          this.isLoading = false;
        }
      }
    },
    loadCity() {
      this.isLoading = true;
      axios.post("/common/get-cities")
          .then((response) => {
            this.cityLists = response.data;
          })
          .catch((error) => {
            this.$notify({
              title: "Error",
              text: "Error while retrieving cities",
              type: "error",
            });
            auth.checkError(error.response.data.message, this.$router, this.$swal);
          })
          .finally(() => {
            this.isLoading = false;
          });
    },
    getDistrictByCity(cityId) {
      this.isLoading = true;
      axios.post(`/common/get-district`, {
        city_id: cityId
      })
          .then((response) => {
            this.districtList = response.data.allDistrict;
            this.selectedDistrict = '';
          })
          .catch((error) => {
            this.$notify({
              title: "Error",
              text: "Error while retrieving districts",
              type: "error",
            });
            auth.checkError(error.response.data.message, this.$router, this.$swal);
          })
          .finally(() => {
            this.isLoading = false;
          });
    },
  },
};
</script>

<style scoped>
.form-control {
  width: 100%;
  padding: 8px;
  font-size: 16px;
  margin-bottom: 16px;
}

.v-text-field__slot {
  margin-bottom: 16px;
}
</style>
