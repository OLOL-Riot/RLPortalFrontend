<script setup lang="ts">
import { getApi } from "@/api";
import CourseDetail from "@/components/CourseDetail.vue";
import type { ICourse, IUpdateCourse } from "@/interfaces/ICourse";
import type { AxiosResponse, AxiosError } from "axios";
import { QPage, useQuasar } from "quasar";
import { ref } from "vue";
import { useRoute } from "vue-router";
import CourseSectionPreview from "@/components/CourseSectionPreview.vue";
import EditCourse from "@/components/forms/EditCourse.vue";
import AddCourseSection from "@/components/forms/AddCourseSection.vue";
import { QDialog } from "quasar";
import EditCourseSection from "@/components/forms/EditCourseSection.vue";
import { isAdministrator } from "@/roles";

const $q = useQuasar();
const route = useRoute();
const courseId = route.params.courseId as string;

const course = ref({} as ICourse);

const load = ref(false);
const editMode = ref(false);
const editSectionMode = ref(false);

const editableSectionId = ref("");
const editableSectionName = ref("");
const editableSectionShortDescription = ref("");
const editableSectionSerialNumber = ref(0);

function getData() {
  getApi().then((api) =>
    api
      .get("api/Course/page/" + courseId)
      .then((response: AxiosResponse<ICourse>) => {
        course.value = response.data;
        course.value.previewCourseSections.sort(
          (a, b) => a.serialNumber - b.serialNumber
        );
        load.value = true;
      })
      .catch((err: AxiosError) => {
        $q.notify({
          color: "red-5",
          textColor: "white",
          icon: "warning",
          message: err.message,
        });
      })
  );
}
getData();

function updateCourse(data: IUpdateCourse) {
  let toSend = data;
  toSend.courseSectionIds = course.value.previewCourseSections.map((e) => e.id);

  getApi().then((api) =>
    api
      .put("api/Course/" + courseId, toSend)
      .then((response: AxiosResponse) => {
        editMode.value = false;
        load.value = false;
        getData();

        $q.notify({
          color: "green-5",
          textColor: "white",
          icon: "done",
        });
      })
      .catch((err: AxiosError) => {
        $q.notify({
          color: "red-5",
          textColor: "white",
          icon: "warning",
          message: err.message,
        });
      })
  );
}

function editSection(sectionId: string) {
  editSectionMode.value = true;

  const editableSection = course.value.previewCourseSections.find(
    (el) => el.id == sectionId
  );

  if (typeof editableSection == "undefined") return;

  editableSectionId.value = editableSection.id;
  editableSectionName.value = editableSection.name;
  editableSectionShortDescription.value = editableSection.shortDescription;
  editableSectionSerialNumber.value = editableSection.serialNumber;
}

function removeSection(sectionId: string) {
  getApi().then((api) =>
    api
      .delete("api/CourseSection/" + sectionId)
      .then((response) => {
        $q.notify({
          color: "green-4",
          textColor: "white",
          icon: "cloud_done",
          message: "???????????????? ????????????????",
        });

        getData();
      })
      .catch((err: AxiosError) => {
        $q.notify({
          color: "red-5",
          textColor: "white",
          icon: "warning",
          message: err.message,
        });
      })
  );
}
</script>

<template>
  <q-page padding v-if="load">
    <CourseDetail
      v-if="!editMode"
      :name="course.name"
      :description="course.description"
      :toEdit="() => (editMode = true)"
    >
      <div class="row">
        <div class="col-12">
          <h3 class="text-h3 q-px-sm q-py-md">???????????? ??????????</h3>
        </div>
        <CourseSectionPreview
          v-for="section in course.previewCourseSections"
          :key="section.id"
          :id="section.id"
          :name="section.name"
          :shortDescription="section.shortDescription"
          :to-edit="editSection"
          :to-remove="removeSection"
        />
        <AddCourseSection
          v-if="isAdministrator()"
          :course-id="course.id"
          :success="getData"
        />
      </div>
    </CourseDetail>
    <EditCourse
      v-else
      :name="course.name"
      :description="course.description"
      :shortDescription="course.shortDescription"
      :onSave="updateCourse"
    />
    <q-dialog v-model="editSectionMode">
      <div>
        <EditCourseSection
          :section-id="editableSectionId"
          :section-name="editableSectionName"
          :section-short-description="editableSectionShortDescription"
          :section-serial-number="editableSectionSerialNumber"
          :success="getData"
        />
      </div>
    </q-dialog>
  </q-page>
</template>
